## Установка и первоначальная настройка КриптоПро УЦ 2.0 для Linux (сборка 1.63.0.32) в среде виртуализации 

Пример установки и первоначальной настройки удостоверяющего центра на базе программного обеспечения от _КриптоПро_. В данном примере попытаемся, используя средства виртуализации, 
развернуть УЦ от [КриптоПро](https://cryptopro.ru/products/ca/2.0) для ознакомления с функционалом и тестирования.
> [!IMPORTANT]
> Сразу отмечу, что рассматриваемая здесь установка _КриптоПро УЦ 2.0_ **не подразумевает** использование его в промышленных целях и не является руководством для 
> подобного применения, так как в данной реализации нет соответствия множеству требованиий, содержащихся в эксплуатационной документации и подлежащих 
> исполнению при работе с _СКЗИ_ со стороны действующего законодательства.

Стенд, на котором будет развернут комплекс, представляет собой хост-машину, где в качестве гипервизора установлено ПО виртуализации _KVM_ и среда разработки - _Vagrant_.
Все узлы _Удостоверяющего Центра_ - это _QEMU_-образы виртуальных машин, которые разворачиваются с помощью средств автоматизации и оркестрации _Vagrant_ и _Ansible_.

### Установка и предварительная настройка виртуальных машин с помощью _Vagrant_

> [!NOTE]
> Создание _Vagrant_-образов виртуальных машин рассматривается в данной [статье](https://github.com/spanishairman/vagrant).
 
Описание виртуальных машин находится в [Vagrantfile](vagrant/Vagrantfile)

В данном _Vagrantfile_ приведено описание трёх виртуальных машин:
  - _cproca.local_ - сервер центра сертификации;
  - _cprora.local_ - сервер центра регистрации;
  - _cprodb.local_ - сервер баз данных соответственно. 
В качестве _СУБД_ используется _PostgreSQL_, установленная из репозиториев операционной системы. 
Данный тестовый _Удостоверяющий центр_ разворачивается в ОС - _Debian Linux 12_. В формуляре же в качестве операционной системы указана Astra linux 1.7.5

Все создаваемые виртуальные машины имеют два сетевых интерфейса: 
  - _vagrant-libvirt-inet1_ - изолированная сеть, без выхода в интернет и доступа к каким-либо хостам, включая хост виртуализации;
  - _vagrant-libvirt-mgmt_ - сеть управления, с её помощью выполняется управление виртуальными машинами, а также эта сеть позволяет виртуальным машинам выходить в интернет. 

### Ansible
#### Файл hosts
Описание виртуальных хостов находится в файле [hosts](vagrant/ansible.ca/staging/hosts)
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
[cprocaserver]
cproca ansible_host=192.168.121.11 ansible_port=22 ansible_private_key_file=/home/max/vagrant/.vagrant/machines/Debian121/libvirt/private_key

[cproraserver]
cprora ansible_host=192.168.121.12 ansible_port=22 ansible_private_key_file=/home/max/vagrant/.vagrant/machines/Debian122/libvirt/private_key

[cprodbserver]
cprodb ansible_host=192.168.121.13 ansible_port=22 ansible_private_key_file=/home/max/vagrant/.vagrant/machines/Debian123/libvirt/private_key

[caservers:children]
cprocaserver
cproraserver

[allservers:children]
cprocaserver
cproraserver
cprodbserver
```

</details>

#### Файлы описания переменных
Описание переменных для серверов УЦ находится в файле [caservers.yml](vagrant/ansible.ca/staging/group_vars/caservers.yml)
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
# --- Global vars ---
allow_world_readable_tmpfiles: true
src_dirinst: /home/max/vagrant/ansible.ca/files
dst_dirinst: /home/vagrant/ansible.ca/files
dst_user: vagrant
localdir: /home/max/vagrant/ansible.ca
# --- CryptoPro CSP ---
liccsp: XXXXX-XXXXX-XXXXX-XXXXX-XXXXX
licocsputils: XXXXX-XXXXX-XXXXX-XXXXX-XXXXX
lictsputils: XXXXX-XXXXX-XXXXX-XXXXX-XXXXX
csp_distro: linux-amd64_deb.tgz
dircsp: csp50r2
cspbase: /opt/cprocsp/
cspbindir: /opt/cprocsp/bin/amd64/
cspsbindir: /opt/cprocsp/sbin/amd64/
# --- PostgreSQL ---
cpca_dbadmin: cpcadbadmin
cpra_dbadmin: cpradbadmin
cpca_db: cpcadb
cpra_db: cpradb
# --- CryptoPro CA/RA ---
admin_acc: cpca-admin
admin_acc_pass: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          31393233363564356530363532313235323262663737323664376233666630353737623766366232
          3464636462653836663134353663303561326335356463370a343963316666356536613334633532
          66326565363862363762393361386166363132303535343763393532663261623732653534373563
          3830303663623331610a346664393233356431316462303762336564356430363536373634326364
          6537
mstore_ca: /var/opt/cprocsp/users/stores/ca.sto
cpca_srv_acc: cpca-srv
cpra_srv_acc: cpra-srv
cabase: /opt/cpca
cadistro: ca-linux-x64-1.63.0.32.zip
casrvbase: /opt/cpca/CryptoPro.Ca.Service
rasrvbase: /opt/cpca/CryptoPro.Ra.Service
rawebbase: /opt/cpca/CryptoPro.Ra.Web
rawebuibase: /opt/cpca/CryptoPro.Ra.Web.Ui
pkicabase: /opt/cpca/pkica
casrvsbin: CryptoPro.Ca.Service
rasrvsbin: CryptoPro.Ra.Service
rawebsbin: CryptoPro.Ra.Web
pkicabin: pkica
natsbase: /opt/cpca/nats-streaming
natssbin: nats-streaming-server
ca_address: 192.168.1.1
ca_hostname: cproca
ra_address: 192.168.1.2
ra_hostname: cprora
pg_address: 192.168.1.3
pg_hostname: cprodb
```

</details>

Описание переменных для сервера баз данных находится в файле [cprodbserver.yml](vagrant/ansible.ca/staging/group_vars/cprodbserver.yml)
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
pg_ver: 15
cpca_dbadmin: cpcadbadmin
cpra_dbadmin: cpradbadmin
cpca_db: cpcadb
cpra_db: cpradb
cpca_dbpass: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          61646162306362363161663530656231303261623738656639303737386565626133393530356562
          6632626563316530636131633264326366623035393166650a316635366531636238376564663831
          39303530313966383431316566666236653563316338373863663063346262616531366663396365
          3866366131643538370a613631346462396237326663353939326166643431323663333461313464
          6561
cpra_dbpass: !vault |
          $ANSIBLE_VAULT;1.1;AES256
          61633765316537343938613931623630633166383664643333643535363935633635616339646636
          3939613062366561313861326337656665306632336666640a396665323235303661613037626334
          39626130653961623536623162376639636431613466616430616232333863636363623266313662
          6630643766623438340a373532356635396438613762323336363435616162373338613436323762
          6133
```
</details>

#### Файл __ansible.cfg__
Конфигурационный файл [ansible.cfg](vagrant/ansible.ca/ansible.cfg)
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
[defaults]
inventory = staging/hosts
remote_user = vagrant
host_key_checking = False
retry_files_enabled = False
```

</details>

#### Установка КриптоПро CSP 5.0 R3
##### Копирование файлов установки на управляемые хосты
Первым делом необходимо загрузить на управляемые машины дистрибутивы программных продуктов, а также, при необходимости, конфигурационные файлы, гамму для подключения ДСЧ КриптоПро исходный материал и прочее.
Загрузка осуществляется с помощью плейбука [01.cproca-copy-distr.yml](vagrant/ansible.ca/play/01.cproca-copy-distr.yml)
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 01 >>> COPY DISTRO | Group of servers "caservers".
  hosts: caservers
  become: false
  tasks:
  - name: COPY.
    ansible.builtin.copy:
      src: "{{ src_dirinst }}/"
      dest: "{{ dst_dirinst }}"
      owner: "{{ dst_user }}"
      group: "{{ dst_user }}"
      mode: '0644'
```

</details>

##### Установка
Далее потребуется распаковать ранее загруженный дистрибутив КриптоПро CSP, запустить вложенный скрипт __install.sh__ с необходимым набором параметров (уровни кс{1,2,3} и набор компонентов), ввести лицензии, добавить гамму и перезапустить службу cprocsp. Все эти шаги выполняет плейбук [02.install-csp.yml](vagrant/ansible.ca/play/02.install-csp.yml)
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 02 >>> INSTALL CSP | Extract Archive.
  hosts: caservers
  tasks:
    - name: INSTALL CSP. Extract Archive.
      ansible.builtin.shell: |
        test -d "{{ dircsp }}" || mkdir $_
        tar -C "{{ dircsp }}" -xvzf {{ csp_distro }} --strip-components 1
      args:
        executable: /bin/bash
        chdir: "{{ dst_dirinst }}/"
      tags:
      - extract

- name: <<< PLAYBOOK 02 >>> INSTALL CSP | Install distributives CryptoPro CSP, Stunnel, Nginx, PKI Cades. Setup Licenses for CSP, OCSP, TSP. Install Gamma.
  hosts: caservers
  become: true
  tasks:
    - name: INSTALL CSP. Install Software CryptoPro CSP, cprocsp-nginx, lsb-cprocsp-devel, cprocsp-stunnel, cprocsp-pki-cades.
      ansible.builtin.shell: |
        ./install.sh kc1 kc2 cprocsp-nginx lsb-cprocsp-devel cprocsp-stunnel cprocsp-pki-cades
      args:
        executable: /bin/bash
        chdir: "{{ dst_dirinst }}/{{ dircsp }}/"
      tags:
      - install

    - name: CONFIGURE CSP. Setup Licenses for CSP, OCSP, TSP.
      ansible.builtin.shell: |
        sbin/amd64/cpconfig -license -set "{{ liccsp }}"
        bin/amd64/ocsputil li -s "{{ licocsputils }}"
        bin/amd64/tsputil li -s "{{ lictsputils }}"
      args:
        executable: /bin/bash
        chdir: "{{ cspbase }}"
      tags:
      - license

    - name: CONFIGURE CSP. Add Gamma.
      ansible.builtin.shell: |
        cp "{{ dst_dirinst }}"/gamma/kis_1 /var/opt/cprocsp/dsrf/db1/
        cp "{{ dst_dirinst }}"/gamma/kis_1 /var/opt/cprocsp/dsrf/db2/
        chmod -R 777 "/var/opt/cprocsp/dsrf/"
        "{{ cspsbindir }}"/cpconfig -hardware rndm -add cpsd -name 'cpsd rng' -level 3
        "{{ cspsbindir }}"/cpconfig -hardware rndm -configure cpsd -add string /db1/kis_1 /var/opt/cprocsp/dsrf/db1/kis_1
        "{{ cspsbindir }}"/cpconfig -hardware rndm -configure cpsd -add string /db2/kis_1 /var/opt/cprocsp/dsrf/db2/kis_1
      args:
        executable: /bin/bash
      tags:
      - gamma

    - name: CONFIGURE CSP. Restart service cprocsp.
      ansible.builtin.service:
        name: cprocsp.service
        state: restarted
      tags:
      - restart

```

</details>

Каждому из перечисленных шагов соответствует уникальный __тэг__:
  - :white_check_mark: extract
  - :white_check_mark: install
  - :white_check_mark: license
  - :white_check_mark: gamma
  - :white_check_mark: restart

#### Установка PostgreSQL Server
Клиентская и серверная часть сервера баз данных устанавливается из репозиториев Debian. Для этого используем плейбук [03.install-postgres.yml](vagrant/ansible.ca/play/03.install-postgres.yml)
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 03 >>> APT. INSTALL POSTGRESQL | Install Client.
  hosts: caservers
  become: true
  tasks:
  - name: INSTALL POSTGRESQL. Install Client.
    ansible.builtin.apt:
      name: acl,postgresql-client,python3-cryptography
      state: present
      update_cache: false

- name: <<< PLAYBOOK 03 >>> APT. INSTALL POSTGRESQL | Install Server.
  hosts: cprodbserver
  become: true
  tasks:
  - name: INSTALL POSTGRESQL. Install Server.
    ansible.builtin.apt:
      name: postgresql,python3-psycopg2,acl
      state: present
      update_cache: false
```

</details>

#### Настройка PostgreSQL Server. Предоставление доступа для внешних подключений
Откроем доступ для подключения к базам данных с удалённых хостов cproca-test и cprora-test. Плейбук [04.postgres-configure.yml](vagrant/ansible.ca/play/04.postgres-configure.yml)
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 04 >>> CONFIGURE POSTGRESQL | Configure Server.
  hosts: cprodbserver
  become: true
  tasks:
  - name: CONFIGURE POSTGRESQL. Configure Server. Edit pg_hba configuration file. Add "{ cproca-test }" and "{ cprora-test }" access
    ansible.builtin.postgresql_pg_hba:
      dest: /etc/postgresql/{{ pg_ver }}/main/pg_hba.conf
      contype: host
      users: "{{ item.users }}"
      source: "{{ item.source }}"
      databases: all
      method: md5
      create: true
    loop:
      - { source: "{{ hostvars['cproca-test'].ansible_host }}", users: "{{ cpca_dbadmin }}" }
      - { source: "{{ hostvars['cprora-test'].ansible_host }}", users: "{{ cpra_dbadmin }}" }

  - name: CONFIGURE POSTGRESQL. Configure Server. Restart service PostgreSQL.
    ansible.builtin.service:
      name: postgresql.service
      state: restarted 
```

</details>

#### Настройка PostgreSQL Server. Создание комплекта служебных баз данных и ролей, настройка привилегий
Создание баз данных и ролей, а так-же, предоставление привилегий для этих ролей на созданных базах. Плейбук [05.postgres-create-dbs-roles.yml](vagrant/ansible.ca/play/05.postgres-create-dbs-roles.yml)
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 05 >>> POSTGRESQL | Configure Server.
  hosts: cprodbserver
  become: true
  become_user: postgres
  tasks:
    - name: PostgreSQL. Configure Server. Create "{{ cpca_dbadmin }}" and "{{ cpra_dbadmin }}" users, and grant access to bases create.
      community.postgresql.postgresql_user:
        name: "{{ item.name }}"
        password: "{{ item.password }}"
        expires: "infinity"
        role_attr_flags: CREATEDB
      loop:
        - { name: "{{ cpca_dbadmin }}", password: "{{ cpca_dbpass }}" }
        - { name: "{{ cpra_dbadmin }}", password: "{{ cpra_dbpass }}" }

    - name: PostgreSQL. Configure Server. Create a new databases "{{ cpca_db }}" and "{{ cpra_db }}".
      community.postgresql.postgresql_db:
        name: "{{ item.name }}"
        owner: "{{ item.owner }}"
      loop:
        - { name: "{{ cpca_db }}", owner: "{{ cpca_dbadmin }}" }
        - { name: "{{ cpra_db }}", owner: "{{ cpra_dbadmin }}" }

    - name: PostgreSQL. Configure Server. Connect to databases "{{ cpca_db }}" and "{{ cpra_db }}", grant privileges on databases objects (database) for roles.
      community.postgresql.postgresql_privs:
        database: "{{ item.database }}"
        privs: "{{ item.privs }}"
        type: "{{ item.type }}"
        roles: "{{ item.roles }}"
        grant_option: "{{ item.grant_option }}"
        state: present
      loop:
        - { database: "{{ cpca_db }}", roles: "{{ cpca_dbadmin }}", privs: 'ALL', type: 'database', grant_option: 'true' }
        - { database: "{{ cpra_db }}", roles: "{{ cpra_dbadmin }}", privs: 'ALL', type: 'database', grant_option: 'true' }

    - name: PostgreSQL. Configure Server. Connect to databases "{{ cpca_db }}" and "{{ cpra_db }}", grant privileges on databases objects (schema) for roles.
      community.postgresql.postgresql_privs:
        database: "{{ item.database }}"
        privs: "{{ item.privs }}"
        type: "{{ item.type }}"
        roles: "{{ item.roles }}"
        objs: "{{ item.objs }}"
        state: present
      loop:
        - { database: "{{ cpca_db }}", roles: "{{ cpca_dbadmin }}", privs: 'CREATE', type: 'schema', objs: 'public' }
        - { database: "{{ cpra_db }}", roles: "{{ cpra_dbadmin }}", privs: 'CREATE', type: 'schema', objs: 'public' }
```
</details>

#### Центр сертификации и Центр регистрации. Создание группы безопасности и служебных пользователей. Настройка файлов аутентификации на сервере баз данных
Для установки CRL в хранилище __LocalMachine\CA__ на серверах Центра Сертификации и Центра Регистрации создается группа __crl-writers__. В эту группу добавляется служебный пользователь, с правами которого работают службы ЦС и ЦР.
Так же, для служебных учетных записей настраивается подключение к базам данных. Плейбук [06.cproca-create-groups-users.yml](vagrant/ansible.ca/play/06.cproca-create-groups-users.yml):
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 06 >>> CPROCA AND CPRORA Servers | Create group crl-writers and set ACLs. Create user with admins privileges
  hosts: caservers
  become: true
  tasks:
    - name: CryptoPro CA. Ensure group "crl-writers" exists
      ansible.builtin.group:
        name: crl-writers
        state: present

    - name: CryptoPro CA. Set ACL privileges for crl-writers group on "{{ mstore_ca }}"
      ansible.posix.acl:
        path: "{{ mstore_ca }}"
        entity: crl-writers
        etype: group
        permissions: rw
        default: false
        recursive: false
        state: present

    - name: CREATE ADMIN ACCOUNT. Add the user "{{ admin_acc }}" with a bash shell and with homedir
      ansible.builtin.user:
        name: "{{ admin_acc }}"
        password: "{{ admin_acc_pass }}"
        comment: CryptoPro CA Admin Account
        shell: /bin/bash
        create_home: true

- name: <<< PLAYBOOK 06 >>> CPROCA | Create CryptoPro Ca service login
  hosts: cprocaserver
  become: true
  tasks:
    - name: CryptoPro CA. Add the user "{{ cpca_srv_acc }}" with a bash shell, appending the group 'crl-writers' and to the user's groups
      ansible.builtin.user:
        name: "{{ cpca_srv_acc }}"
        password: '!'
        comment: CryptoPro CA Service
        shell: /bin/bash
        create_home: yes
        groups: crl-writers
        append: yes

    - name: CryptoPro CA. Create .pgpass file for "{{ cpca_srv_acc }}"
      ansible.builtin.shell: |
        echo '# host:port:database:user:password' > /home/"{{ cpca_srv_acc }}"/.pgpass
        echo "{{ pg_address }}:5432:*:{{ cpca_dbadmin }}:{{ cpca_dbpass }}" >> /home/"{{ cpca_srv_acc }}"/.pgpass
        echo "{{ pg_address }}:5432:*:{{ cpra_dbadmin }}:{{ cpra_dbpass }}" >> /home/"{{ cpca_srv_acc }}"/.pgpass
      args:
        executable: /bin/bash

    - name: Cproca. Chmod .pgpass
      ansible.builtin.file:
        path: "/home/{{ cpca_srv_acc }}/.pgpass"
        owner: "{{ cpca_srv_acc }}"
        group: "{{ cpca_srv_acc }}"
        mode: '0600'

- name: <<< PLAYBOOK 06 >>> CPRORA | Create user CryptoPro Ra service login
  hosts: cproraserver
  become: true
  tasks:
    - name: CryptoPro RA. Add the user "{{ cpra_srv_acc }}" with a bash shell
      ansible.builtin.user:
        name: "{{ cpra_srv_acc }}"
        password: '!'
        comment: CryptoPro RA Service
        shell: /bin/bash
        create_home: yes
        groups: crl-writers
        append: yes

    - name: CryptoPro RA. Create .pgpass file for "{{ cpra_srv_acc }}"
      ansible.builtin.shell: |
        echo "# host:port:database:user:password" > /home/"{{ cpra_srv_acc }}"/.pgpass
        echo "{{ pg_address }}:5432:*:{{ cpra_dbadmin }}:{{ cpra_dbpass }}" >> /home/"{{ cpra_srv_acc }}"/.pgpass
        echo "{{ pg_address }}:5432:*:{{ cpca_dbadmin }}:{{ cpca_dbpass }}" >> /home/"{{ cpra_srv_acc }}"/.pgpass
      args:
        executable: /bin/bash

    - name: Cprora. Chmod .pgpass
      ansible.builtin.file:
        path: "/home/{{ cpra_srv_acc }}/.pgpass"
        owner: "{{ cpra_srv_acc }}"
        group: "{{ cpra_srv_acc }}"
        mode: '0600'
```
</details>

#### Центр сертификации и Центр регистрации. Распаковка дистрибутива, копирование файлов в рабочую директорию. Установка разрешений
Установка службы Центра Сертификации, как и Центра Регистрации - это простое копирование содержимого архива ca-linux-x64-x.xx.x.xx.zip в каталог /opt/cpca.

После копирования необходимо дать сервисным учетным записям __cpca-srv__ и __cpra-srv__ права на запуск соответствующих исполняемых файлов - **CryptoPro.Ca.Service**, **CryptoPro.Ra.Service** и **CryptoPro.Ra.Web**. 
Плейбук [07.cproca-extract-distro-set-permissions.yml](vagrant/ansible.ca/play/07.cproca-extract-distro-set-permissions.yml):
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 07 >>> CPROCA and CPRORA | Install package CPCA.
  hosts: caservers
  become: true
  tasks:
    - name: CryptoPro CA. APT. Install package "unzip" to latest version
      ansible.builtin.apt:
        name: unzip
        state: present
        update_cache: false

    - name: CryptoPro CA. Extract archive.
      ansible.builtin.shell: |
        test -d "{{ cabase }}" || mkdir $_
        unzip -o -d "{{ cabase }}" "{{ cadistro }}"
      args:
        executable: /bin/bash
        chdir: "{{ dst_dirinst }}"

- name: <<< PLAYBOOK 07 >>> CPROCA | Set permissions and ACL privileges for CryptoPro.Ca.Service and Pkica util
  hosts: cprocaserver
  become: true
  tasks:
    - name: CryptoPro CA. Chmod {{ cabase }}
      ansible.builtin.file:
        path: "{{ cabase }}"
        group: "{{ cpca_srv_acc }}"
        mode: o-rwx
        recurse: true

    - name: CryptoPro CA. Set ACL privileges for PkiCA, CryptoPro.Ca.Service, Nats-Streaming
      ansible.posix.acl:
        path: "{{ item.path }}"
        entity: "{{ item.entity }}"
        etype: "{{ item.etype }}"
        permissions: "{{ item.permissions }}"
        default: "{{ item.default }}"
        recursive: "{{ item.recursive }}"
        state: "{{ item.state }}"
      loop:
        - { path: "{{ casrvbase }}/{{ casrvsbin }}", entity: "{{ cpca_srv_acc }}", etype: 'user', permissions: rx, default: false, recursive: false, state: present }
        - { path: "{{ pkicabase }}/{{ pkicabin }}", entity: "{{ cpca_srv_acc }}", etype: 'user', permissions: rx, default: false, recursive: false, state: present }
        - { path: "{{ pkicabase }}/{{ pkicabin }}", entity: "{{ admin_acc }}", etype: 'user', permissions: rx, default: false, recursive: false, state: present }
        - { path: "{{ natsbase }}/{{ natssbin }}", entity: "{{ cpca_srv_acc }}", etype: 'user', permissions: rx, default: false, recursive: false, state: present }

- name: <<< PLAYBOOK 07 >>> CPRORA | Set permissions and ACL privileges for CryptoPro.Ra.Service and Pkica util
  hosts: cproraserver
  become: true
  tasks:
    - name: CryptoPro RA. Chmod {{ cabase }}
      ansible.builtin.file:
        path: "{{ cabase }}"
        group: "{{ cpra_srv_acc }}"
        mode: o-rwx
        recurse: true
    - name: CryptoPro RA. Set ACL privileges for CryptoPro.Ra.Service, CryptoPro.Ra.Web
      ansible.posix.acl:
        path: "{{ item.path }}"
        entity: "{{ item.entity }}"
        etype: "{{ item.etype }}"
        permissions: "{{ item.permissions }}"
        default: "{{ item.default }}"
        recursive: "{{ item.recursive }}"
        state: "{{ item.state }}"
      loop:
        - { path: "{{ rasrvbase }}/{{ rasrvsbin }}", entity: "{{ cpra_srv_acc }}", etype: 'user', permissions: rx, default: false, recursive: false, state: present }
        - { path: "{{ rawebbase }}/{{ rawebsbin }}", entity: "{{ cpra_srv_acc }}", etype: 'user', permissions: rx, default: false, recursive: false, state: present }
        - { path: "{{ pkicabase }}/{{ pkicabin }}", entity: "{{ cpra_srv_acc }}", etype: 'user', permissions: rx, default: false, recursive: false, state: present }
        - { path: "{{ pkicabase }}/{{ pkicabin }}", entity: "{{ admin_acc }}", etype: 'user', permissions: rx, default: false, recursive: false, state: present }

```
</details>

#### Установка параметров для служб Центра сертификации и Центра регистрации
Для утилиты настройки Удостоверяющего Центра - __Pkica__, сервиса **Центра Сертификации** и сервиса **Центра Регистрации** основные настройки находятся в файлах __appsettings.json__, располагающихся в соответствующих каталогах:

- /opt/cpca/pkica/appsettings.json - файл конфигурации pkica - программы настройки УЦ;
- /opt/cpca/CryptoPro.Ca.Service/appsettings.json - файл конфигурации CryptoPro.Ca.Service - сервиса ЦС
- /opt/cpca/CryptoPro.Ra.Service/appsettings.json - файл конфигурации CryptoPro.Ra.Service - сервиса ЦР.

С помощью следующих плейбуков каждый файл в этих каталогах приводится в актуальное для работы состояние:
 - play/08.1.cproca-config-distros-set-parameters-pkica.yml - задаёт параметры подключения к базам данных и опции шифрования для утилиты __pkica__;
 - play/08.2.cproca-config-distros-set-parameters-ca.yml;
 - play/08.3.cproca-config-distros-set-parameters-ra.yml;
 - play/08.4.cproca-config-distros-set-parameters-raweb.yml.

Плейбук [play/08.1.cproca-config-distros-set-parameters-pkica.yml](vagrant/ansible.ca/play/08.1.cproca-config-distros-set-parameters-pkica.yml):
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
---
- name: <<< PLAYBOOK 08.1 >>> CPROCA AND CPRORA | Config pkica appsettings.json.
  hosts: caservers
  become: true
  tasks:
    - name: CryptoPro CA and RA. Edit main config for pkica (pkica - Программа настройки УЦ)
      ansible.builtin.shell: |
        sed -i "/CaDb/{n;n;s/Server=localhost;Database=Ca;Username=postgres;Pooling=True/Server={{ pg_address }};Database={{ cpca_db }};Username={{ cpca_dbadmin }};Pooling=True/}" appsettings.json
        sed -i "/CertRegistryDb/{n;n;s/Server=localhost;Database=CertRegistry;Username=postgres;Pooling=True/Server={{ pg_address }};Database={{ certreg_db }};Username={{ cpca_dbadmin }};Pooling=True/}" appsettings.json
        sed -i "/RaDb/{n;n;s/Server=localhost;Database=Ra;Username=postgres;Pooling=True/Server={{ pg_address }};Database={{ cpra_db }};Username={{ cpra_dbadmin }};Pooling=True/}" appsettings.json
        sed -i 's/"Secure": true/"Secure": false/' appsettings.json
        sed -i "/Nats/{n;s/localhost/{{ ca_hostname }}/}" appsettings.json
        sed -i "/Stan/{n;s/localhost/{{ ca_hostname }}/}" appsettings.json
      args:
        executable: /bin/bash
        chdir: "{{ pkicabase }}"
```
</details>

Здесь мы на серверах __ЦС__ и __ЦР__ в файле настроек __appsettings.json__ службы __Pkica__: 
  - :white_check_mark: установили адреса подключения к базам данных:
    - :heavy_check_mark: Центра сертификации - __cpcadb__, 
    - :heavy_check_mark: Реестра сертификатов - __CertRegistry__, 
    - :heavy_check_mark: Центра регистрации - __cpradb__;
  - :white_check_mark: отключили опцию шифрования при подключении к службе очередей __Nats__, 
  - :white_check_mark: задали адреса для служб: 
    - :white_check_mark: __Nats__,
    - :white_check_mark: __Stan__. 

При этом, все изменения производились с помощью модуля __shell__ и текстового процессора __sed__ - т.е. использовался т.н. _bashsible_. 

Плейбук [play/08.2.cproca-config-distros-set-parameters-ca.yml](vagrant/ansible.ca/play/08.2.cproca-config-distros-set-parameters-ca.yml):
<details> 
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 08.2 >>> CPROCA | Config appsettings.json.
  hosts: cprocaserver
  become: true
  tasks:
    - name: CryptoPro CA. Edit main config (CryptoPro.Ca.Service - Сервис ЦС)
      ansible.builtin.shell: |
        sed -i "s/Server=localhost;Database=Ca;Username=postgres;Pooling=True/Server={{ pg_address }};Database={{ cpca_db }};Username={{ cpca_dbadmin }};Pooling=True/" appsettings.json
        sed -i '/nats/{n;n;s/"Secure": true/"Secure": false/}' appsettings.json
        sed -i '/ClientID/{n;n;s/"Secure": true/"Secure": false/}' appsettings.json
        sed -i "/Nats/{n;s/localhost/$HOSTNAME/}" appsettings.json
        sed -i "/Stan/{n;s/localhost/$HOSTNAME/}" appsettings.json
        sed -i "s/\"SerialNumber\": \"\"/\"SerialNumber\": \"{{ licca }}\"/" appsettings.json
        sed -i 's/"Company": ""/"Company": "{{ company }}"/' appsettings.json
      args:
        executable: /bin/bash
        chdir: "{{ casrvbase }}"

    - name: CryptoPro CA. Edit nats-streaming-server daemon config (nats-streaming-server - Служба очередей NATS Streaming с поддержкой ГОСТ TLS)
      ansible.builtin.shell: |
        sed -i "s/ca.example/$HOSTNAME/" nats.no-tls.conf
        sed -i "s/ca.example/{{ inventory_hostname }}/" nats.conf
      args:
        executable: /bin/bash
        chdir: "{{ natsbase }}"
```
</details>

Здесь мы для службы __CryptoPro CA__ отредактировали строку подключения к базе данных Центра сертификации, отключили шифрование для подключения к службе __Nats__, ввели серийный номер и название компании.
Так же, изменили адрес, на котором работает служба __nats-streaming-server__.

Плейбук [play/08.3.cproca-config-distros-set-parameters-ra.yml](vagrant/ansible.ca/play/08.3.cproca-config-distros-set-parameters-ra.yml):
<details> 
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 08.3 >>> CPRORA | Config appsettings.json
  hosts: cproraserver
  become: true
  tasks:
    - name: CryptoPro RA. Edit main config (CryptoPro.Ra.Service - Сервис ЦР)
      ansible.builtin.shell: |
        sed -i "s/Server=localhost;Database=Ra;Username=postgres;Pooling=True/Server={{ pg_address }};Database={{ cpra_db }};Username={{ cpra_dbadmin }};Pooling=True/" appsettings.json
        sed -i '/nats/{n;n;s/"Secure": true/"Secure": false/}' appsettings.json
        sed -i '/ClientID/{n;n;s/"Secure": true/"Secure": false/}' appsettings.json
        sed -i "/Nats/{n;s/localhost/{{ ca_hostname }}/}" appsettings.json
        sed -i "/Stan/{n;s/localhost/{{ ca_hostname }}/}" appsettings.json
        sed -i 's/"PublishCrls": false/"PublishCrls": true/' appsettings.json
        sed -i 's/"PublishCaCerts": false/"PublishCaCerts": true/' appsettings.json
        sed -i "s/\"SerialNumber\": \"\"/\"SerialNumber\": \"{{ licra }}\"/" appsettings.json
        sed -i 's/"Company": ""/"Company": "{{ company }}"/' appsettings.json
      args:
        executable: /bin/bash
        chdir: "{{ rasrvbase }}"
```
</details>

Здесь выполняются те же действия, что и в предыдущем плейбуке, но для службы __CryptoPro.Ra.Service__, дополнительно задаются настройки для публикации списков отзыва и сертификата УЦ.

Плейбук [play/08.4.cproca-config-distros-set-parameters-raweb.yml](vagrant/ansible.ca/play/08.4.cproca-config-distros-set-parameters-raweb.yml):
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 08.4 >>> CPRORA | Config appsettings.json
  hosts: cproraserver
  become: true
  tasks:
    - name: CryptoPro RA. Edit main config (CryptoPro.Ra.Web - Веб интерфейс RA)
      ansible.builtin.shell: |
        sed -i "s/Server=localhost;Database=Ra;Username=postgres;Pooling=True/Server={{ pg_address }};Database={{ cpra_db }};Username={{ cpra_dbadmin }};Pooling=True/" appsettings.json
        sed -i '/nats/{n;n;s/"Secure": true/"Secure": false/}' appsettings.json
        sed -i '/ClientID/{n;n;s/"Secure": true/"Secure": false/}' appsettings.json
        sed -i "s/localhost:5000/$HOSTNAME:5000/" appsettings.json
        sed -i "/Nats/{n;s/localhost/{{ ca_hostname }}/}" appsettings.json
        sed -i "/Stan/{n;s/localhost/{{ ca_hostname }}/}" appsettings.json
        sed -i "s/\"SerialNumber\": \"\"/\"SerialNumber\": \"{{ licra }}\"/" appsettings.json
        sed -i 's/"Company": ""/"Company": "{{ company }}"/' appsettings.json
      args:
        executable: /bin/bash
        chdir: "{{ rawebbase }}"

    - name: CryptoPro RA. Edit main config (CryptoPro.Ra.Web.UI - Веб интерфейс RA)
      ansible.builtin.shell: |
        sed -i "s/ra.example/$HOSTNAME/" init.json
      args:
        executable: /bin/bash
        chdir: "{{ rawebuibase }}"
```
</details>

Здесь задаются настройки веб-служб __Центра регистрации__.
