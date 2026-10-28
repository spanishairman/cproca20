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
  - _Debian121_ - сервер Центра сертификации;
  - _Debian122_ - сервер Центра регистрации;
  - _Debian123_ - сервер баз данных соответственно. 
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
ca_domain: local
aiastore: /var/lib/cpca/aia
cdpstore: /var/lib/cpca/cdp
# --- Certificate and Templates ---
tpl_tlssrv: "1.2.643.2.2.46.0.3"
tpl_tlsclnt: "1.2.643.2.2.46.0.6"
tpl_oper: "1.2.643.2.2.46.0.8"
crt_ra_srv: rasrv
crt_ra_clnt: raclnt
crt_ra_webclnt: webclnt
crt_ra_pkicaclnt: pkiraclnt
crt_ca_natsclnt: natsclnt
crt_ca_natssrv: natssrv
crt_ca_caclnt: caclnt
crt_ca_pkicaclnt: pkicaclnt
crt_opertls: opertls
crt_opersign: opersign
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

### Копирование файлов установки на управляемые хосты
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


### Установка КриптоПро CSP 5.0 R3

Далее потребуется распаковать ранее загруженный дистрибутив КриптоПро CSP, запустить вложенный скрипт __install.sh__ с необходимым набором параметров (уровни кс{1,2} и набор компонентов), ввести лицензии, добавить гамму и перезапустить службу cprocsp. Все эти шаги выполняет плейбук [02.install-csp.yml](vagrant/ansible.ca/play/02.install-csp.yml)
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
        ./install.sh kc1 lsb-cprocsp-devel cprocsp-pki-cades
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
        cp "{{ dst_dirinst }}"/Gamma/db1/kis_1 /var/opt/cprocsp/dsrf/db1/
        cp "{{ dst_dirinst }}"/Gamma/db2/kis_1 /var/opt/cprocsp/dsrf/db2/
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

### Установка PostgreSQL Server
Клиентская и серверная часть Postgresql устанавливается из репозиториев Debian. Для этого используем плейбук [03.install-postgres.yml](vagrant/ansible.ca/play/03.install-postgres.yml)
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

В данном плейбуке мы для серверов __cproca__ и __cprora__ (группа хостов __caservers__ в файле __staging/hosts__) установили пакеты:
  - :white_check_mark: acl
  - :white_check_mark: postgresql-client
  - :white_check_mark: python3-cryptography

Для сервера __cprodbserver__ же, были установлены:
  - :white_check_mark: acl
  - :white_check_mark: postgresql
  - :white_check_mark: python3-psycopg2


### Настройка PostgreSQL Server. Предоставление доступа для внешних подключений
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
      source: "{{ item.source }}/32"
      databases: all
      method: md5
      create: true
    loop:
      - { source: "{{ ca_address }}", users: "{{ cpca_dbadmin }}" }
      - { source: "{{ ra_address }}", users: "{{ cpra_dbadmin }}" }

  - name: CONFIGURE POSTGRESQL. Configure Server. Edit postgresql.conf configuration file. Set listen_addresses = '*'
    ansible.builtin.lineinfile:
      path: /etc/postgresql/{{ pg_ver }}/main/postgresql.conf
      line: "listen_addresses = '*'"
      insertafter: "#listen_addresses = 'localhost'"
      firstmatch: yes
      state: present

  - name: CONFIGURE POSTGRESQL. Configure Server. Restart service PostgreSQL.
    ansible.builtin.service:
      name: postgresql.service
      state: restarted
```

</details>

Данный плейбук выполняет редактирование файла __pg_hba__, а именно открывает следующие доступы ко всем базам данных:
  - :white_check_mark: с хоста __cproca__ для пользователя __cpca\_dbadmin__
  - :white_check_mark: с хоста __cprora__ для пользователя __cpra\_dbadmin__

### Настройка PostgreSQL Server. Создание комплекта баз данных и ролей, настройка привилегий
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

    - name: PostgreSQL. Configure Server. Create a new databases "{{ cpca_dbadmin }}" and "{{ cpra_dbadmin }}".
      community.postgresql.postgresql_db:
        name: "{{ item.name }}"
        owner: "{{ item.owner }}"
      loop:
        - { name: "{{ cpca_dbadmin }}", owner: "{{ cpca_dbadmin }}" }
        - { name: "{{ cpra_dbadmin }}", owner: "{{ cpra_dbadmin }}" }

    - name: PostgreSQL. Configure Server. Connect to databases "{{ cpca_dbadmin }}" and "{{ cpra_dbadmin }}", grant privileges on databases objects (database) for roles.
      community.postgresql.postgresql_privs:
        database: "{{ item.database }}"
        privs: "{{ item.privs }}"
        type: "{{ item.type }}"
        roles: "{{ item.roles }}"
        grant_option: "{{ item.grant_option }}"
        state: present
      loop:
        - { database: "{{ cpca_dbadmin }}", roles: "{{ cpca_dbadmin }}", privs: 'ALL', type: 'database', grant_option: 'true' }
        - { database: "{{ cpra_dbadmin }}", roles: "{{ cpra_dbadmin }}", privs: 'ALL', type: 'database', grant_option: 'true' }

    - name: PostgreSQL. Configure Server. Connect to databases "{{ cpca_dbadmin }}" and "{{ cpra_dbadmin }}", grant privileges on databases objects (schema) for roles.
      community.postgresql.postgresql_privs:
        database: "{{ item.database }}"
        privs: "{{ item.privs }}"
        type: "{{ item.type }}"
        roles: "{{ item.roles }}"
        objs: "{{ item.objs }}"
        state: present
      loop:
        - { database: "{{ cpca_dbadmin }}", roles: "{{ cpca_dbadmin }}", privs: 'CREATE', type: 'schema', objs: 'public' }
        - { database: "{{ cpra_dbadmin }}", roles: "{{ cpra_dbadmin }}", privs: 'CREATE', type: 'schema', objs: 'public' }


```
</details>

Здесь мы на сервере баз данных:
  - :white_check_mark: создали роли:
    - :heavy_check_mark: __cpca_dbadmin__ - администратор базы данных Центра сертификации;
    - :heavy_check_mark: __cpra_dbadmin__ - администратор базы данных Центра регистрации;
  - :white_check_mark: создали базы данных:
    - :heavy_check_mark: __cpca_dbadmin__ - владелец - __cpca_dbadmin__;
    - :heavy_check_mark: __cpra_dbadmin__ - владелец - __cpra_dbadmin__;
  - :white_check_mark: предоставили привилегии __ALL__ на базы данных:
    - :heavy_check_mark: __cpca_dbadmin__ - для роли __cpca_dbadmin__;
    - :heavy_check_mark: __cpra_dbadmin__ - для роли __cpra_dbadmin__;
  - :white_check_mark: предоставили привилегии __CREATE__ на тип __schema__ объекта __public__ баз данных:
    - :heavy_check_mark: __cpca_dbadmin__ - для роли __cpca_dbadmin__;
    - :heavy_check_mark: __cpra_dbadmin__ - для роли __cpra_dbadmin__;

> [!IMPORTANT]
> Рабочий комплект баз данных _Центра сертификации_, _Центра регистрации_ и _Реестра сертификатов_ будет создан позднее при помощи утилиты _pkica_.
> А в данном случае были созданы роли и одноименные базы данных по причине того, что они требуются для работы этой утилиты с сервером баз данных. 
> Если вместо служебных ролей __cpca_dbadmin__ и ___cpra_dbadmin__ в настройках подключения использовать встроенную роль _Postgres_, то данный шаг выполнять не требуется. 

### Центр сертификации и Центр регистрации. Создание группы безопасности и служебных пользователей. Настройка файлов аутентификации на сервере баз данных
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

### Центр сертификации и Центр регистрации. Распаковка дистрибутива, копирование файлов в рабочую директорию. Установка разрешений
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
    - name: CryptoPro RA. Install Nginx
      ansible.builtin.apt:
        deb: "{{ nginxbase }}/{{ nginxdistro }}"
```
</details>

### Установка параметров для служб Центра сертификации и Центра регистрации
Для утилиты настройки Удостоверяющего Центра - __Pkica__, сервиса **Центра Сертификации** и сервиса **Центра Регистрации** основные настройки находятся в файлах __appsettings.json__, располагающихся в соответствующих каталогах:

- /opt/cpca/pkica/appsettings.json - файл конфигурации __pkica__ - программы настройки УЦ;
- /opt/cpca/CryptoPro.Ca.Service/appsettings.json - файл конфигурации __CryptoPro.Ca.Service__ - сервиса ЦС
- /opt/cpca/CryptoPro.Ra.Service/appsettings.json - файл конфигурации __CryptoPro.Ra.Service__ - сервиса ЦР.

#### Способ 1. Использование для редактирования конфигурационных файлов  ansible-модуля shell и текстового процессора sed

С помощью следующих плейбуков каждый файл в этих каталогах приводится в актуальное для работы состояние:
 - _play/08.1.cproca-config-distros-set-parameters-pkica.yml_ - задаёт параметры подключения к базам данных и опции шифрования для утилиты __pkica__;
 - _play/08.2.cproca-config-distros-set-parameters-ca.yml_ - задаёт параметры подключения к базе данных Центра сертификации, опции шифрования при подключении к __Nats__, адреса __Nats__ и __Stan__, параметры лицензии, а так-же параметры в файлах конфигурации службы __Nats__;
 - _play/08.3.cproca-config-distros-set-parameters-ra.yml_ - то же самое, но для Центра регистрации;
 - _play/08.4.cproca-config-distros-set-parameters-raweb.yml_ - то же самое, но для Веб службы Центра регистрации.

Плейбук [play/08.1.cproca-config-distros-set-parameters-pkica.yml](vagrant/ansible.ca/play/08.1.cproca-config-distros-set-parameters-pkica.yml):
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
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
    - :heavy_check_mark: __Nats__,
    - :heavy_check_mark: __Stan__. 


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

Здесь мы для служб: 
  - :white_check_mark: __CryptoPro CA__ на сервере Центра сертификации:
    - :heavy_check_mark: отредактировали строку подключения к базе данных Центра сертификации; 
    - :heavy_check_mark: отключили шифрование для подключения к службе __Nats__;
    - :heavy_check_mark: отключили шифрование для подключения к службе __Stan__;
    - :heavy_check_mark: указали в качестве адреса для подключения к службе __Nats__ собственное имя хоста вместо __localhost__;
    - :heavy_check_mark: указали в качестве адреса для подключения к службе __Stan__ собственное имя хоста вместо __localhost__;
    - :heavy_check_mark: ввели серийный номер и название компании;
  - :white_check_mark: __nats-streaming-server__:
    - :heavy_check_mark: изменили адрес, на котором работает служба.

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

Здесь для службы __CryptoPro.Ra.Service__ на сервере Центра регистрации: 
  - :white_check_mark: отредактировали строку подключения к базе данных Центра регистрации;
  - :white_check_mark: отключили шифрование для подключения к службе __Nats__;
  - :white_check_mark: отключили шифрование для подключения к службе __Stan__;
  - :white_check_mark: указали в качестве адреса для подключения к службе __Nats__ имя хоста центра сертификации вместо __localhost__;
  - :white_check_mark: указали в качестве адреса для подключения к службе __Stan__ имя хоста центра сертификации вместо __localhost__;
  - :white_check_mark: включили публикацию списков отзыва;
  - :white_check_mark: включили публикацию сертификтов ЦС;
  - :white_check_mark: ввели серийный номер и название компании.


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

Здесь для службы __CryptoPro.Ra.Web.Service__ на сервере Центра регистрации: 
  - :white_check_mark: отредактировали строку подключения к базе данных Центра регистрации;
  - :white_check_mark: отключили шифрование для подключения к службе __Nats__;
  - :white_check_mark: отключили шифрование для подключения к службе __Stan__;
  - :white_check_mark: изменили адрес для подключения к службе с _localhost_ на собственное имя хоста;
  - :white_check_mark: указали в качестве адреса для подключения к службе __Nats__ имя хоста центра сертификации вместо __localhost__;
  - :white_check_mark: указали в качестве адреса для подключения к службе __Stan__ имя хоста центра сертификации вместо __localhost__;
  - :white_check_mark: ввели серийный номер и название компании.

#### Способ 2. Использование ansible-модуля lineinfile

В предыдущем параграфе все изменения производились с помощью модуля __shell__ и текстового процессора __sed__ - т.е. использовался т.н. _bashsible_. 

Здесь же, для получения того же результата, к которому приводит работа плейбуков 8.{1-4}, мы будем пользоваться модулем __lineinfile__. 

Следующий плейбук [play/09.1.cproca-config-distros-set-parameters-pkica.yml](vagrant/ansible.ca/play/09.1.cproca-config-distros-set-parameters-pkica.yml) выполняет установку нужных параметров в файле __appsettings.json__ утилиты __pkica__:

<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 09.1 >>> CPROCA AND CPRORA | Config "ConnectionString" parameter for Pkica appsettings.json.
  hosts: caservers
  become: true
  tasks:
    - name: CryptoPro CA. Edit pkica appsettings.json.
      ansible.builtin.lineinfile:
        backup: "{{ item.backup }}"
        path: "{{ item.path }}"
        search_string: "{{ item.ss }}"
        firstmatch: false
        line: '    "ConnectionString": "Server={{ item.server }};Database={{ item.database }};Username={{ item.username }};Pooling=True"'
      loop:
        - { backup: true,  path: "{{ pkicabase }}/appsettings.json", ss: "Database=Ca", server: "{{ pg_address }}", database: "{{ cpca_db }}", username: "{{ cpca_dbadmin }}" }
        - { backup: false, path: "{{ pkicabase }}/appsettings.json", ss: "Database=CertRegistry", server: "{{ pg_address }}", database:  "{{ certreg_db }}", username: "{{ cpca_dbadmin }}" }
        - { backup: false, path: "{{ pkicabase }}/appsettings.json", ss: "Database=Ra", server: "{{ pg_address }}", database: "{{ cpra_db }}", username: "{{ cpra_dbadmin }}" }

    - name: CryptoPro CA. Edit pkica appsettings.json.
      ansible.builtin.lineinfile:
        backup: "{{ item.backup }}"
        path: "{{ item.path }}"
        search_string: "{{ item.search }}"
        firstmatch: "{{ item.fm }}"
        line: '{{ item.line }}'
      loop:
        - { backup: false, path: "{{ pkicabase }}/appsettings.json", search: "Secure", fm: true, line: '    "Secure": false, // включить TLS' }
        - { backup: false, path: "{{ pkicabase }}/appsettings.json", search: "Url", fm: true, line: '    "Url": "nats://{{ ca_hostname }}:4222",' }
        - { backup: false, path: "{{ pkicabase }}/appsettings.json", search: "Secure", fm: false, line: '    "Secure": false, // включить TLS' }
        - { backup: false, path: "{{ pkicabase }}/appsettings.json", search: "Url", fm: false, line: '    "Url": "nats://{{ ca_hostname }}:4222",' }

```
</details>

Здесь мы на серверах __ЦС__ и __ЦР__ в файле настроек __appsettings.json__ службы __Pkica__:
  - :white_check_mark: установили адреса подключения к базам данных:
    - :heavy_check_mark: Центра сертификации - __cpcadb__,
    - :heavy_check_mark: Реестра сертификатов - __CertRegistry__,
    - :heavy_check_mark: Центра регистрации - __cpradb__;
  - :white_check_mark: отключили опцию шифрования при подключении к службе очередей __Nats__,
  - :white_check_mark: задали адрес для службы __Nats__,
  - :white_check_mark: отключили опцию шифрования при подключении к службе очередей __Stan__,
  - :white_check_mark: задали адрес для службы __Stan__,

Следующий плейбук [play/09.2.cproca-config-distros-set-parameters-ca.yml](vagrant/ansible.ca/play/09.2.cproca-config-distros-set-parameters-ca.yml) выполняет установку нужных параметров в файле __appsettings.json__ Центра сертификации и файлах __nats.conf__, __nats.no-tls.conf__ Службы очередей _NATS Streaming_:
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 09.2 >>> CPROCA | Set parameters for CA appsettings.json.
  hosts: cprocaserver
  become: true
  tasks:
    - name: CryptoPro CA. Edit CA appsettings.json. Config "ConnectionString" parameter.
      ansible.builtin.lineinfile:
        backup: "{{ item.backup }}"
        path: "{{ item.path }}"
        search_string: "{{ item.ss }}"
        firstmatch: false
        line: '    "ConnectionString": "Server={{ item.server }};Database={{ item.database }};Username={{ item.username }};Pooling=True"'
      loop:
        - { backup: true, path: "{{ casrvbase }}/appsettings.json", ss: "Database=Ca", server: "{{ pg_address }}", database: "{{ cpca_db }}", username: "{{ cpca_dbadmin }}" }

    - name: CryptoPro CA. Edit CA appsettings.json. Config "Secure, Url, SN, Company" parameters
      ansible.builtin.lineinfile:
        backup: "{{ item.backup }}"
        path: "{{ item.path }}"
        search_string: "{{ item.search }}"
        firstmatch: "{{ item.fm }}"
        line: '{{ item.line }}'
      loop:
        - { backup: false, path: "{{ casrvbase }}/appsettings.json", search: "Secure", fm: true, line: '    "Secure": false, // включить TLS' }
        - { backup: false, path: "{{ casrvbase }}/appsettings.json", search: "Url", fm: true, line: '    "Url": "nats://{{ ca_hostname }}:4222",' }
        - { backup: false, path: "{{ casrvbase }}/appsettings.json", search: "Secure", fm: false, line: '    "Secure": false, // включить TLS' }
        - { backup: false, path: "{{ casrvbase }}/appsettings.json", search: "Url", fm: false, line: '    "Url": "nats://{{ ca_hostname }}:4222",' }
        - { backup: false, path: "{{ casrvbase }}/appsettings.json", search: "SerialNumber", fm: false, line: '    "SerialNumber": "{{ licca }}",' }
        - { backup: false, path: "{{ casrvbase }}/appsettings.json", search: "Company", fm: false, line: '    "Company": "{{ company }}"' }

    - name: CryptoPro CA. Edit nats-streaming-server daemon config (nats-streaming-server - Служба очередей NATS Streaming с поддержкой ГОСТ TLS). Config "Listen" parameter
      ansible.builtin.lineinfile:
        backup: "{{ item.backup }}"
        path: "{{ item.path }}"
        search_string: "{{ item.search }}"
        firstmatch: "{{ item.fm }}"
        line: '{{ item.line }}'
      loop:
        - { backup: true, path: "{{ natsbase }}/nats.conf", search: "listen:", fm: true, line: 'listen:         {{ inventory_hostname }}:4222' }
        - { backup: true, path: "{{ natsbase }}/nats.no-tls.conf", search: "listen", fm: true, line: 'listen:         {{ inventory_hostname }}:4222' }
```
</details>

Здесь мы для служб:
  - :white_check_mark: __CryptoPro CA__ на сервере Центра сертификации:
    - :heavy_check_mark: отредактировали строку подключения к базе данных Центра сертификации;
    - :heavy_check_mark: отключили шифрование для подключения к службе __Nats__;
    - :heavy_check_mark: указали в качестве адреса для подключения к службе __Nats__ собственное имя хоста вместо __localhost__;
    - :heavy_check_mark: отключили шифрование для подключения к службе __Stan__;
    - :heavy_check_mark: указали в качестве адреса для подключения к службе __Stan__ собственное имя хоста вместо __localhost__;
    - :heavy_check_mark: ввели серийный номер и название компании;
  - :white_check_mark: __nats-streaming-server__ - Служба очередей NATS Streaming с поддержкой ГОСТ TLS:
    - :heavy_check_mark: изменили адрес, на котором работает эта служба.

Плейбук [play/09.3.cprora-config-distros-set-parameters-ra.yml](vagrant/ansible.ca/play/09.3.cprora-config-distros-set-parameters-ra.yml) выполняет установку нужных параметров в файле __appsettings.json__ Центра регистрации:
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 09.3 >>> CPRORA | Set parameters for RA appsettings.json.
  hosts: cproraserver
  become: true
  tasks:
    - name: CryptoPro RA. Edit RA appsettings.json. Config "ConnectionString" parameter.
      ansible.builtin.lineinfile:
        backup: "{{ item.backup }}"
        path: "{{ item.path }}"
        search_string: "{{ item.ss }}"
        firstmatch: false
        line: '    "ConnectionString": "Server={{ item.server }};Database={{ item.database }};Username={{ item.username }};Pooling=True"'
      loop:
        - { backup: true, path: "{{ rasrvbase }}/appsettings.json", ss: "Database=Ra", server: "{{ pg_address }}", database: "{{ cpra_db }}", username: "{{ cpra_dbadmin }}" }

    - name: CryptoPro RA. Edit RA appsettings.json. Config "Secure, Url, SN, Company" parameters
      ansible.builtin.lineinfile:
        backup: "{{ item.backup }}"
        path: "{{ item.path }}"
        search_string: "{{ item.search }}"
        firstmatch: "{{ item.fm }}"
        line: '{{ item.line }}'
      loop:
        - { backup: false, path: "{{ rasrvbase }}/appsettings.json", search: "Secure", fm: true, line: '    "Secure": false, // включить TLS' }
        - { backup: false, path: "{{ rasrvbase }}/appsettings.json", search: "Url", fm: true, line: '    "Url": "nats://{{ ca_hostname }}:4222",' }
        - { backup: false, path: "{{ rasrvbase }}/appsettings.json", search: "Secure", fm: false, line: '    "Secure": false, // включить TLS' }
        - { backup: false, path: "{{ rasrvbase }}/appsettings.json", search: "Url", fm: false, line: '    "Url": "nats://{{ ca_hostname }}:4222",' }
        - { backup: false, path: "{{ rasrvbase }}/appsettings.json", search: "PublishCrls", fm: false, line: '    "PublishCrls": true,' }
        - { backup: false, path: "{{ rasrvbase }}/appsettings.json", search: '"NodeId":', fm: true, line: '    "NodeId": "{{ inventory_hostname }}"' }
        - { backup: false, path: "{{ rasrvbase }}/appsettings.json", search: "PublishCaCerts", fm: false, line: '    "PublishCaCerts": true,' }
        - { backup: false, path: "{{ rasrvbase }}/appsettings.json", search: '"NodeId":', fm: false, line: '    "NodeId": "{{ inventory_hostname }}"' }
        - { backup: false, path: "{{ rasrvbase }}/appsettings.json", search: "SerialNumber", fm: false, line: '    "SerialNumber": "{{ licra }}",' }
        - { backup: false, path: "{{ rasrvbase }}/appsettings.json", search: "Company", fm: false, line: '    "Company": "{{ company }}"' }
```
</details>

Здесь для службы __CryptoPro.Ra.Service__ на сервере Центра регистрации:
  - :white_check_mark: отредактировали строку подключения к базе данных Центра регистрации;
  - :white_check_mark: отключили шифрование для подключения к службе __Nats__;
  - :white_check_mark: указали в качестве адреса для подключения к службе __Nats__ имя хоста центра сертификации вместо __localhost__;
  - :white_check_mark: отключили шифрование для подключения к службе __Stan__;
  - :white_check_mark: указали в качестве адреса для подключения к службе __Stan__ имя хоста центра сертификации вместо __localhost__;
  - :white_check_mark: включили публикацию списков отзыва;
  - :white_check_mark: указали уникальный NodeID для публикации списков отзыва (требуется для работы в кластере);
  - :white_check_mark: включили публикацию сертификтов ЦС;
  - :white_check_mark: указали уникальный NodeID для публикации сертификатов ЦС (требуется для работы в кластере);
  - :white_check_mark: ввели серийный номер и название компании.

Плейбук [play/09.4.cprora-config-distros-set-parameters-raweb.yml](vagrant/ansible.ca/play/09.4.cprora-config-distros-set-parameters-raweb.yml) выполняет установку нужных параметров в файле __appsettings.json__ Веб-службы Центра регистрации:
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 09.4 >>> CPRORA | Set parameters for RAWEB appsettings.json.
  hosts: cproraserver
  become: true
  tasks:
    - name: CryptoPro RA. Edit RAWEB appsettings.json. Config "ConnectionString" parameter.
      ansible.builtin.lineinfile:
        backup: "{{ item.backup }}"
        path: "{{ item.path }}"
        search_string: "{{ item.ss }}"
        firstmatch: false  
        line: '    "ConnectionString": "Server={{ item.server }};Database={{ item.database }};Username={{ item.username }};Pooling=True"'
      loop:
        - { backup: true, path: "{{ rawebbase }}/appsettings.json", ss: "Database=Ra", server: "{{ pg_address }}", database: "{{ cpra_db }}", username: "{{ cpra_dbadmin }}" }

    - name: CryptoPro RA. Edit RAWEB appsettings.json. Config "Secure, Url, SN, Company" parameters
      ansible.builtin.lineinfile:
        backup: "{{ item.backup }}"
        path: "{{ item.path }}"
        search_string: "{{ item.search }}"
        firstmatch: "{{ item.fm }}"
        line: '{{ item.line }}'
      loop:
        - { backup: false, path: "{{ rawebbase }}/appsettings.json", search: "localhost:5000", fm: false, line: '  "Urls": "http://{{ inventory_hostname }}:5000",' }
        - { backup: false, path: "{{ rawebbase }}/appsettings.json", search: "Secure", fm: true, line: '    "Secure": false, // включить TLS' }
        - { backup: false, path: "{{ rawebbase }}/appsettings.json", search: '"Url":', fm: true, line: '    "Url": "nats://{{ ca_hostname }}:4222",' }
        - { backup: false, path: "{{ rawebbase }}/appsettings.json", search: "Secure", fm: false, line: '    "Secure": false, // включить TLS' }
        - { backup: false, path: "{{ rawebbase }}/appsettings.json", search: '"Url":', fm: false, line: '    "Url": "nats://{{ ca_hostname }}:4222",' }
        - { backup: false, path: "{{ rawebbase }}/appsettings.json", search: "SerialNumber", fm: false, line: '    "SerialNumber": "{{ licra }}",' }
        - { backup: false, path: "{{ rawebbase }}/appsettings.json", search: "Company", fm: false, line: '    "Company": "{{ company }}"' }

```
</details>

Здесь для службы __CryptoPro.Ra.Web.Service__ на сервере Центра регистрации:
  - :white_check_mark: отредактировали строку подключения к базе данных Центра регистрации;
  - :white_check_mark: изменили адрес для подключения к службе с _localhost_ на собственное имя хоста;
  - :white_check_mark: отключили шифрование для подключения к службе __Nats__;
  - :white_check_mark: указали в качестве адреса для подключения к службе __Nats__ имя хоста центра сертификации вместо __localhost__;
  - :white_check_mark: отключили шифрование для подключения к службе __Stan__;
  - :white_check_mark: указали в качестве адреса для подключения к службе __Stan__ имя хоста центра сертификации вместо __localhost__;
  - :white_check_mark: ввели серийный номер и название компании.

### Создание рабочих каталогов для служб Центра Сертификации, Nats-Streaming и Центра регистрации

Здесь мы создаём следующие каталоги:

  - /var/lib/nats-streaming/pkica-store - каталог хранения сообщений;
  - /opt/cpca/nats-streaming/ssl - каталог хранения серверных и клиентских сертификатов и ключей Nats;
  - /var/log/cpca - хранение журналов Центра Сертификации и Центра регистрации;
  - /var/lib/cpca/cdp - каталог хранения списков отзыва на Центре регистрации;
  - /var/lib/cpca/aia - хранилище сертификатов УЦ на Центре регистрации;

Назначаем владелца - учетную запись службы Центра сертификации или службы Центра регистрации в зависимости от роли, выполняемой виртуальной машиной.

Плейбук [play/10.create-folders.yml](vagrant/ansible.ca/play/10.create-folders.yml):
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 10 >>> CPROCA. CREATE FOLDERS SET OWNS | Create a messages store dir for nats-streaming-server. Create directories.
  hosts: cprocaserver
  become: true
  tasks:
    - name: CREATE FOLDERS SET OWNS. Create a messages, certificates store dirs for nats-streaming-server, log store dir for ca-server
      ansible.builtin.file:
        path: "{{ item.path }}"
        state: directory
        owner: "{{ item.ownname }}"
      loop:
        - { path: "{{ natslib }}/pkica-store", ownname: "{{ cpca_srv_acc }}" }
        - { path: "{{ natsbase }}/ssl", ownname: "{{ cpca_srv_acc }}" }
        - { path: "{{ calog }}", ownname: "{{ cpca_srv_acc }}" }

- name: <<< PLAYBOOK 09 >>> CPRORA. CREATE FOLDERS SET OWNS | Create directories
  hosts: cproraserver
  become: true
  tasks:
    - name: CREATE FOLDERS SET OWNS. Create a crl, crt, log store dirs for ra-server
      ansible.builtin.file:
        path: "{{ item.path }}"
        state: directory
        owner: "{{ item.ownname }}"
      loop:
        - { path: "{{ cdpstore }}", ownname: "{{ cpra_srv_acc }}" }
        - { path: "{{ aiastore }}", ownname: "{{ cpra_srv_acc }}" }
        - { path: "{{ calog }}", ownname: "{{ cpra_srv_acc }}" }

```
</details> 

Здесь на Центре сертификации создаются:

  - :white_check_mark: /var/lib/nats-streaming/pkica-store - каталог хранения сообщений;
  - :white_check_mark: /opt/cpca/nats-streaming/ssl - каталог хранения серверных и клиентских сертификатов и ключей Nats;
  - :white_check_mark: /var/log/cpca - каталог для хранения журналов Центра Сертификации.

На Центре регистрации:
  - :white_check_mark: /var/lib/cpca/cdp - каталог хранения списков отзыва;
  - :white_check_mark: /var/lib/cpca/aia - хранилище сертификатов УЦ;
  - :white_check_mark: /var/log/cpca - каталог для хранения журналов Центра Регистрации.

Для всех созданных каталогов задаются владелец и разрешения. 

### Postgresql. Создание служебных баз данных с помощью утилиты _pkica_
Служебные базы данных Центра сертификации, Центра регистрации и Реестра сертификатов создаются при помощи утилиты _pkica_. Плейбук [play/11.cproca-create-db.yml](vagrant/ansible.ca/play/11.cproca-create-db.yml).
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 11 >>> CPROCA | Create databases for CA service.
  hosts: cprocaserver
  become: true
  become_user: "{{ cpca_srv_acc }}"
  tasks:
    - name: CryptoPro CA. Create databases for CA service
      ansible.builtin.shell: |
        yes | ./"{{ pkicabin }}" {{ item.db }} db new
      args:
        executable: /bin/bash
        chdir: "{{ pkicabase }}"
      loop:
        - { db: ca }
        - { db: cert-registry }

- name: <<< PLAYBOOK 11 >>> CPRORA | Create databases for RA service.
  hosts: cproraserver
  become: true
  become_user: "{{ cpra_srv_acc }}"
  tasks:
    - name: CryptoPro RA. Create database for RA service
      ansible.builtin.shell: |
        yes | ./"{{ pkicabin }}" ra db new
      args:
        executable: /bin/bash
        chdir: "{{ pkicabase }}"

```
</details>

Здесь на Центре сертификации запускаются команды для создания:

  - :white_check_mark: cpcadb - база данный Центра сертификации;
  - :white_check_mark: CertRegistry - база данных Реестра сертификатов;

На Центре регистрации запускается команда для создания:
  - :white_check_mark: cpradb - база данный Центра регистрации;.

### Создание файлов служб и их регистрация
#### Создание файлов служб CryptoPro.Ca.Service, nats-streaming-server, CryptoPro.Ra.Service, CryptoPro.Ra.Web.Service
Для создания файлов _.service_ на удалённых хостах будем использовать шаблоны _jinja2_. Плейбук для создания файлов и их копирования на удалённые серверы [play/12.services-creation.yml](vagrant/ansible.ca/play/12.services-creation.yml): 
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 12 >>> REGISTRATION OF SERVICES | Server cpcaserver, services CryptoPro.Ca.Service, Nats-Streaming-Server
  hosts: cproca
  become: true
  tasks:
    - name: REGISTRATION OF SERVICE CryptoPro.Ca.Service. Copy Template
      vars:
        description: 'CryptoPro.Ca.Service'
        folder: 'CryptoPro.Ca.Service'
        service: 'CryptoPro.Ca.Service'
        identifier: 'cryptopro-ca-service'
        owner: 'cpca-srv'
        key: ''
      ansible.builtin.template:
        src: services.j2
        dest: /etc/systemd/system/cryptopro.ca.service

    - name: REGISTRATION OF SERVICE Nats-Streaming-Server. Copy Template
      vars:
        description: 'nats-streaming-server'
        folder: 'nats-streaming'
        service: 'nats-streaming-server'
        identifier: 'nats-streaming-server'
        owner: 'cpca-srv'
        key: '-sc /opt/cpca/nats-streaming/nats.no-tls.conf'
      ansible.builtin.template:
        src: services.j2
        dest: /etc/systemd/system/nats-streaming-server.service


- name: <<< PLAYBOOK 12 >>> REGISTRATION OF SERVICES | Server cpraserver, services CryptoPro.Ra.Service, CryptoPro.Ra.Web.Service
  hosts: cprora
  become: true
  tasks:
    - name: REGISTRATION OF SERVICE CryptoPro.Ra.Service. Copy Template
      vars:
        description: 'CryptoPro.Ra.Service'
        folder: 'CryptoPro.Ra.Service'
        service: 'CryptoPro.Ra.Service'
        identifier: 'cryptopro-ra-service'
        owner: 'cpra-srv'
        key: ''
      ansible.builtin.template:
        src: services.j2
        dest: /etc/systemd/system/cryptopro.ra.service

    - name: REGISTRATION OF SERVICE CryptoPro.Ra.Web.Service. Copy Template
      vars:
        description: 'CryptoPro.Ra.Web.Service'
        folder: 'CryptoPro.Ra.Web'
        service: 'CryptoPro.Ra.Web'
        identifier: 'cryptopro-ra-web-service'
        owner: 'cpra-srv'
        key: ''
      ansible.builtin.template:
        src: services.j2
        dest: /etc/systemd/system/cryptopro.ra.web.service

```
</details>

Здесь мы на сервере Центра сертификации, используя шаблон _jinja2_, создали два файла служб в каталоге _/etc/systemd/system_:
  - :white_check_mark: _cryptopro.ca.service_;
  - :white_check_mark: _nats-streaming-server.service_.

На сервере Центра регистрации, в каталоге _/etc/systemd/system_, так же создали два файла служб:
  - :white_check_mark: _cryptopro.ra.service_;
  - :white_check_mark: _cryptopro.ra.web.service_.

Шаблон для создания файлов служб [play/templates/services.j2](vagrant/ansible.ca/play/templates/services.j2):

<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
[Unit]
Description={{ description }}
[Service]
WorkingDirectory=/opt/cpca/{{ folder }}/
ExecStart=/opt/cpca/{{ folder }}/{{ service }} {{ key }}
Restart=always
# Restart service after 10 seconds if the dotnet service crashes:
RestartSec=10
KillSignal=SIGINT
SyslogIdentifier={{ identifier }}
User={{ owner }}
[Install]
WantedBy=multi-user.target

```
</details>

#### Регистрация служб _cryptopro.ca.service_, _nats-streaming-server.service_, _cryptopro.ra.service_, _cryptopro.ra.web.service_

Плейбук [play/13.services-registration.yml](vagrant/ansible.ca/play/13.services-registration.yml):
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 13 >>> REGISTRATION OF SERVICES | Server cproca
  hosts: cproca
  become: true
  tasks:
    - name: Force systemd to reread configs
      ansible.builtin.systemd_service:
        daemon_reload: true

    - name: REGISTRATION OF SERVICES. Enable and start services
      ansible.builtin.service:
        name: "{{ item.name }}"
        enabled: true
        state: started
      loop:
        - { name: cryptopro.ca.service }
        - { name: nats-streaming-server.service }

- name: <<< PLAYBOOK 13 >>> REGISTRATION OF SERVICES | Server cprora
  hosts: cprora
  become: true
  tasks:
    - name: Force systemd to reread configs
      ansible.builtin.systemd_service:
        daemon_reload: true

    - name: REGISTRATION OF SERVICES. Enable and start services
      ansible.builtin.service:
        name: "{{ item.name }}"
        enabled: true
        state: started
      loop:
        - { name: cryptopro.ra.service }
        - { name: cryptopro.ra.web.service }

```
</details>

Здесь мы на сервере Центра сертификации, зарегистрировали и запустили службы:
  - :white_check_mark: _cryptopro.ca.service_;
  - :white_check_mark: _nats-streaming-server.service_.

На сервере Центра регистрации так же зарегистрировали и запустили две службы:
  - :white_check_mark: _cryptopro.ra.service_;
  - :white_check_mark: _cryptopro.ra.web.service_.

### Создание экземпляра Центра сертификации и загрузка шаблонов по умолчанию

Теперь нужно создать новый экземпляр Центра сертификации с именем "Тестовый Удостоверяющий Центр АО GAMMA-TRADE".
Для этого используется плейбук [play/14.cproca-create-instance-load-tpls.yml](vagrant/ansible.ca/play/14.cproca-create-instance-load-tpls.yml):
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 14 >>> CPROCA | Create CA instance. Load Templates
  hosts: cproca
  become: true
  become_user: "{{ cpca_srv_acc }}"
  tasks:
    - name: CryptoPro CA. Create CA instance
      ansible.builtin.shell: |
        ./"{{ pkicabin }}" ca authority new --name "{{ caname }}"
      args:
        executable: /bin/bash
        chdir: "{{ pkicabase }}"

    - name: CryptoPro CA. Add templates
      ansible.builtin.shell: |
        ./"{{ pkicabin }}" ca template add --default-template "{{ item.template }}"
      args:
        executable: /bin/bash
        chdir: "{{ pkicabase }}"
      loop:
        - { template: Tls }
        - { template: User }
```
</details>

В данном примере был создан новый экземпляр ЦС и загружены два шаблона сертифкатов по умолчанию:
  - :white_check_mark: Tls - сертификат TLS-сервера;
  - :white_check_mark: User - сертификат пользователя.

### Выпуск и установка самоподписанного сертификата
Плейбук [play/15.cproca-release-install-ca-cert.yml](vagrant/ansible.ca/play/15.cproca-release-install-ca-cert.yml):
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 15 >>> CPROCA | Release, Install, Add a new CA certificate
  hosts: cproca
  become: true
  become_user: "{{ cpca_srv_acc }}"
  tasks:
    - name: CPROCA. Release a new CA certificate
      ansible.builtin.shell: |
        ./"{{ pkicabin }}" ca new-root-cert --out-file ~/{{ cacert }}.crt --key-index 0 --subject "CN={{ caname }};O={{ orgname }};C=RU" --container-name {{ cacont }} --no-pin --silent
      args:
        executable: /bin/bash
        chdir: "{{ pkicabase }}"
      tags:
        - release

    - name: CPROCA. Install CA certificates to storages
      ansible.builtin.shell: |
        ./certmgr -install -store uMy -file ~/"{{ cacert }}.crt" -provtype 80 -container "{{ cacont }}" -at_signature
        yes o | ./certmgr -install -store uRoot -file ~/"{{ cacert }}.crt"
      args:
        executable: /bin/bash
        chdir: "{{ cspbindir }}"
      tags:
        - install

    - name: CPROCA. Add CA key
      ansible.builtin.shell: |
        export PATH=$PATH:"{{ cspbindir }}"
        ./"{{ pkicabin }}" ca signing-key add --name "{{ caname }}" --thumbprint $(certmgr -list -file ~/"{{ cacert }}.crt" | grep SHA1 | awk '{print $4}') --store-name My --store-location CurrentUser --no-pin
      args:
        executable: /bin/bash
        chdir: "{{ pkicabase }}"
      tags:
        - export
```
</details>

Здесь мы:
  - :white_check_mark: создали ключ и сертификат корневого ЦС;
  - :white_check_mark: установили сертификат ЦС в хранилище My пользователя, от имени которого запускается сервис ЦС;
  - :white_check_mark: установили сертификат ЦС в хранилище Root пользователя, от имени которого запускается сервис ЦС;
  - :white_check_mark: добавили ключ подписи ЦС экземпляру ЦС, чтобы использовать его для подписи сертификатов и CRL.

### Изменение настроек Центра сертификации
В этой главе нам нужно задать расписние выпуска списков отзыва, экспортировать настройки в текстовый файл в формате _json_, отредактировать в нём часть параметров и импортировать новые настройки из этого файла.
Плейбук [play/16.cproca-configure-ca.yml](vagrant/ansible.ca/play/16.cproca-configure-ca.yml):
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 16 >>> CPROCA | Configure CA and CA Shedule
  hosts: cproca
  become: true
  become_user: "{{ cpca_srv_acc }}"
  tasks:
    - name: CryptoPro CA. Configure Settings for CA and CA Shedule. Sheduling and Export
      ansible.builtin.shell: |
        ./"{{ pkicabin }}" ca crl schedule update --name "{{ caname }}" --base "0 0 18 * * *" --delta -
        ./"{{ pkicabin }}" ca authority export-settings --name "{{ caname }}" --file ~/"{{ casettings }}" --format Json
      args:
        executable: /bin/bash
        chdir: "{{ pkicabase }}"
      tags: export

    - name: CryptoPro CA. Configure Settings for CA and CA Shedule. Edit
      ansible.builtin.lineinfile:
        path: ~/{{ casettings }}
        backup: "{{ item.backup }}"
        search_string: "{{ item.ss }}"
        line: "{{ item.line }}"
      loop:
        - { backup: true, ss: "DisableAkiIssuerNameAndSerial", line: '"DisableAkiIssuerNameAndSerial": true,' }
        - { backup: false, ss: "QualifiedCertMode", line: '"QualifiedCertMode": "AttrFormat",' }
        - { backup: false, ss: "AutoLoadKey", line: '"AutoLoadKey": true,' }
        - { backup: false, ss: "CrlPublicationUrls", line: '"CrlPublicationUrls": [ { "Usage": "CdpIncludeInCert", "Urls": [ "http://{{ ra_hostname }}/cdp/%17.crl" ] } ],' }
        - { backup: false, ss: "CaCertPublicationUrls", line: '"CaCertPublicationUrls": [ { "Usage": "AiaIncludeInCert", "Url": "http://{{ ra_hostname }}/aia/{{ cacert }}" } ],' }

    - name: CryptoPro CA. Configure Settings for CA and CA Shedule. Import
      ansible.builtin.shell: |
        ./"{{ pkicabin }}" ca authority import-settings --name "{{ caname }}" --file ~/"{{ casettings }}" --format Json
      args:
        executable: /bin/bash
        chdir: "{{ pkicabase }}"

- name: <<< PLAYBOOK 16 >>> CPROCA | Restart CA
  hosts: cproca
  become: true
  tasks:
    - name: CryptoPro CA. Restart service.
      ansible.builtin.service:
        name: cryptopro.ca.service
        state: restarted

```
</details>

Здесь мы:
  - :white_check_mark: задали расписание для выпуска списков отзыва каждые 18 часов, при этом отключили выпуск разностного списка отзыва;
  - :white_check_mark: экспортировали настройки по умолчанию с действуещего Центра сертификации в текстовый _json_-файл;
  - :white_check_mark: при помощи модуля _lineinfile_ отредактировали в полученном файле следующие параметры:
    - :heavy_check_mark: "DisableAkiIssuerNameAndSerial": _true_ - отключили расширение «Идентификатор ключа центра сертификатов» (Authority Key Identifier, OID 2.5.29.35);
    - :heavy_check_mark: "QualifiedCertMode": "AttrFormat" -  формат атрибутов имени в субъекте проверяется только при их наличии (ИНН, ИНН ЮЛ, СНИЛС, ОГРН, ОГРНИП);
    - :heavy_check_mark: "AutoLoadKey": _true_ - разрешить автозагрузку ключей для экземпляра ЦС;
    - :heavy_check_mark: "CrlPublicationUrls" - указали включать адрес точки распространения _CRL_ в расширение «Точки распространения списков отзыва (CRL)» сертификата;
    - :heavy_check_mark: "CaCertPublicationUrls" - указали включать адрес точек доступа к информации о ЦС в расширение «Доступ к сведениям Центра сертификации»;
  - :white_check_mark: импортировали новые настройки из файла в Центр сертификации;
  - :white_check_mark: перезапустили службу ЦС для применения новых параметров.

### Создание запросов на выпуск сертификатов на Центре Регистрации

На сервере Центра Регистрации выполняются запросы на выпуск сертификатов для:
  - Сервер Центра Регистрации;
  - Клиент Центра Регистрации;
  - Клиент Веб-службы Центра Регистрации;
  - Оператор (TLS);
  - Оператор (Sign).

Плейбук [play/17.cprora-certs-req.yml](vagrant/ansible.ca/play/17.cprora-certs-req.yml):
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 17 >>> CPRORA | Creating a Certificate Issue Request for RA Server, RA Client, RA Web Client
  hosts: cproraserver
  become: true
  become_user: "{{ cpra_srv_acc }}"
  tasks:
    - name: CPORA. Creating a Certificate Issue Request for RA Server, RA Client, RA Web Client
      ansible.builtin.shell: |
        test -f {{ item.file }} && ./"{{ pkicabin }}" ca new-cert-req --subject "{{ item.subject }}" --key-spec KeyExchange --container-name {{ item.container }} --template {{ item.template }} --out-file {{ item.file }} {{ item.additional }} --use-existing-key
        test -f {{ item.file }} || ./"{{ pkicabin }}" ca new-cert-req --subject "{{ item.subject }}" --key-spec KeyExchange --container-name {{ item.container }} --template {{ item.template }} --out-file {{ item.file }} {{ item.additional }}
      args:
        executable: /bin/bash
        chdir: "{{ pkicabase }}"
      loop:
        - { subject: "CN={{ inventory_hostname }}, C=RU, 1.2.643.2.2.44.5={{ inventory_hostname }}", container: "{{ crt_ra_srv }}_cont", template: "{{ tpl_tlssrv }}", file: "{{ ansible_user_dir }}/{{ crt_ra_srv }}.req" , additional: "--no-pin --exportable" }
        - { subject: "CN=ra, C=RU", container: "{{ crt_ra_clnt }}_cont", template: "{{ tpl_tlsclnt }}", file: "{{ ansible_user_dir }}/{{ crt_ra_clnt }}.req" , additional: "--no-pin --exportable" }
        - { subject: "CN=ra, C=RU", container: "{{ crt_ra_webclnt }}_cont", template: "{{ tpl_tlsclnt }}", file: "{{ ansible_user_dir }}/{{ crt_ra_webclnt }}.req" , additional: "--no-pin --exportable" }
        - { subject: "CN=pkica, C=RU", container: "{{ crt_ra_pkicaclnt }}_cont", template: "{{ tpl_tlsclnt }}", file: "{{ ansible_user_dir }}/{{ crt_ra_pkicaclnt }}.req" , additional: "--no-pin --exportable" }

    - name: CryptoPro RA. Creating a Certificate Issue Request for RA Operator TLS, RA Operator Sign
      ansible.builtin.shell: |
        test -f {{ item.file }} && ./"{{ pkicabin }}" ca new-cert-req --subject "{{ item.subject }}" --key-spec KeyExchange --container-name {{ item.container }} --template {{ item.template }} --out-file {{ item.file }} {{ item.additional }} --use-existing-key
        test -f {{ item.file }} || ./"{{ pkicabin }}" ca new-cert-req --subject "{{ item.subject }}" --key-spec KeyExchange --container-name {{ item.container }} --template {{ item.template }} --out-file {{ item.file }} {{ item.additional }}
      args:
        executable: /bin/bash
        chdir: "{{ pkicabase }}"
      loop:
        - { subject: "CN={{ crt_opertls }}, C=RU", container: "{{ crt_opertls }}_cont", template: "{{ tpl_tlsclnt }}", file: "{{ ansible_user_dir }}/{{ crt_opertls }}.req" , additional: "--no-pin --exportable" }
        - { subject: "CN={{ crt_opersign }}, C=RU", container: "{{ crt_opersign }}_cont", template: "{{ tpl_oper }}", file: "{{ ansible_user_dir }}/{{ crt_opersign }}.req" , additional: "--no-pin --exportable" }

```
</details>

В данном примере на сервере Центра регистрации были выполнены запросы на выпуск сертификатов для:

  - :white_check_mark: веб сервера Центра регистрации с использованием шаблона __"1.2.643.2.2.46.0.3"__;
  - :white_check_mark: службы Центра регистрации для подключения к _Nats_ с использованием шаблона __"1.2.643.2.2.46.0.6"__;
  - :white_check_mark: веб службы Центра регистрации для подключения к _Nats_ с использованием шаблона __"1.2.643.2.2.46.0.6"__;
  - :white_check_mark: утилиты _pkica_Центра регистрации для подключения к _Nats_ с использованием шаблона __"1.2.643.2.2.46.0.6"__;
  - :white_check_mark: оператора Центра регистрации для создания защищённого подключения с использованием шаблона __"1.2.643.2.2.46.0.6"__;
  - :white_check_mark: оператора Центра регистрации для подписания запросов на выпуск сертификатов с использованием шаблона __"1.2.643.2.2.46.0.8"__.

### Создание запросов на выпуск сертификатов на Центре Сертификации

На Центре Сертификации создаются запросы на выпуск сертификатов для:
  - Nats-Server - для работы серверной части службы очередей _Nats_;
  - Nats-Client - для клиентского подключения к серверной части службы очередей _Nats_ со стороны её клиентской части;
  - PKICA-Client - для подключения к службе очередей _Nats_ со стороны утилиты _Pkica_;
  - CA-Client - для подключения к службе очередей со стороны службы _Cryptopro.Ca_.

Плейбук [play/18.cproca-certs-req.yml](vagrant/ansible.ca/play/18.cproca-certs-req.yml):

<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 18 >>> CPROCA | Creating a Certificate Issue Request for NATS Server, NATS Client, PKICA Client, CA Client
  hosts: cprocaserver
  become: true
  become_user: "{{ cpca_srv_acc }}"
  tasks:
    - name: CPORA. Creating a Certificate Issue Request for NATS Server, NATS Client, PKICA Client, CA Client
      ansible.builtin.shell: |
        test -f {{ item.file }} && ./{{ pkicabin }} ca new-cert-req --subject "{{ item.subject }}" --key-spec "KeyExchange" --container-name {{ item.container }} --template "{{ item.template }}" --out-file {{ item.file }} {{ item.additional }} --use-existing-key
        test -f {{ item.file }} || ./{{ pkicabin }} ca new-cert-req --subject "{{ item.subject }}" --key-spec "KeyExchange" --container-name {{ item.container }} --template "{{ item.template }}" --out-file {{ item.file }} {{ item.additional }}
      args:
        executable: /bin/bash
        chdir: "{{ pkicabase }}"
      loop:
        - { subject: "CN={{ inventory_hostname }}, C=RU, 1.2.643.2.2.44.5={{ inventory_hostname }}.{{ ca_domain }}", container: "{{ crt_ca_natssrv }}_cont", template: "{{ tpl_tlssrv }}", file: "{{ ansible_user_dir }}/{{ crt_ca_natssrv }}.req" , additional: "--no-pin --exportable" }
        - { subject: "CN=nats, C=RU", container: "{{ crt_ca_natsclnt }}_cont", template: "{{ tpl_tlsclnt }}", file: "{{ ansible_user_dir }}/{{ crt_ca_natsclnt }}.req" , additional: "--no-pin --exportable" }
        - { subject: "CN=pkica, C=RU", container: "{{ crt_ca_pkicaclnt }}_cont", template: "{{ tpl_tlsclnt }}", file: "{{ ansible_user_dir }}/{{ crt_ca_pkicaclnt }}.req" , additional: "--no-pin --exportable" }
        - { subject: "CN=ca, C=RU", container: "{{ crt_ca_caclnt }}_cont", template: "{{ tpl_tlsclnt }}", file: "{{ ansible_user_dir }}/{{ crt_ca_caclnt }}.req" , additional: "--no-pin --exportable" }

```
</details>

В данном примере на сервере Центра регистрации были выполнены запросы на выпуск сертификатов для:

  - :white_check_mark: серверной части службы очередей _Nats_ с использованием шаблона __"1.2.643.2.2.46.0.3"__;
  - :white_check_mark: клиентской части службы очередей _Nats_ для подключения к её серверной компоненте с использованием шаблона __"1.2.643.2.2.46.0.6"__;
  - :white_check_mark: утилиты _Pkica_ Центра сертификации для подключения к _Nats_ с использованием шаблона __"1.2.643.2.2.46.0.6"__;
  - :white_check_mark: службы Центра сертификации для подключения к _Nats_ с использованием шаблона __"1.2.643.2.2.46.0.6"__;

### Копирование полученных запросов на выпуск сертификатов с ЦР на ЦС
Для передачи запросов на выпуск сертификатов задействуется управляющая машина. Таким образом, файлы запросов сначала скачиваются на хост управления с Центра Регистрации, 
затем загружаются с него на Центр Сертификации. С помощью плейбука [play/19.cprora-reqs-fetch.yml](vagrant/ansible.ca/play/19.cprora-reqs-fetch.yml) скопируем все ранее созданные запросы 
на хост управления в каталог _share_:
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 19 >>> FETCH REQUESTS | Download requests for certificates from RA.
  hosts: cprora
  become: true
  become_user: "{{ cpra_srv_acc }}"
  tasks:
    - name: FETCH.
      ansible.builtin.fetch:
        src: "{{ ansible_user_dir }}/{{ item.file }}.req"
        dest: "{{ localdir }}/share/"
        flat: 'true'
      loop:
        - { file: "{{ crt_ra_srv }}" }
        - { file: "{{ crt_ra_clnt }}" }
        - { file: "{{ crt_ra_webclnt }}" }
        - { file: "{{ crt_ra_pkicaclnt }}" }
        - { file: "{{ crt_opertls }}" }
        - { file: "{{ crt_opersign }}" }
```
</details>

Затем загрузим запросы из каталога _share_ на Центр сертификации. Плейбук [play/20.cproca-reqs-up.yml](vagrant/ansible.ca/play/20.cproca-reqs-up.yml):
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>
 
```
---
- name: <<< PLAYBOOK 20 >>> UP REQUESTS | Upload requests for certificates to CA.
  hosts: cproca
  become: true
  become_user: "{{ cpca_srv_acc }}"
  tasks:
    - name: UP.
      ansible.builtin.copy:
        src: "{{ localdir }}/share/{{ item.file }}.req"
        dest: "{{ ansible_user_dir }}/{{ item.file }}.req"
      loop:
        - { file: "{{ crt_ra_srv }}" }
        - { file: "{{ crt_ra_clnt }}" }
        - { file: "{{ crt_ra_webclnt }}" }
        - { file: "{{ crt_ra_pkicaclnt }}" }
        - { file: "{{ crt_opertls }}" }
        - { file: "{{ crt_opersign }}" }
```
</details>

В данном разделе мы скачали на управляющий хост в каталог _share_ с Центра регистрации, а затем загрузили на Центр сертификации следующие запросы на выпуск сертификатов:

  - :white_check_mark: веб сервера Центра регистрации;
  - :white_check_mark: службы Центра регистрации для подключения к _Nats_;
  - :white_check_mark: веб службы Центра регистрации для подключения к _Nats_;
  - :white_check_mark: утилиты _pkica_Центра регистрации для подключения к _Nats_;
  - :white_check_mark: оператора Центра регистрации для создания защищённого подключения;
  - :white_check_mark: оператора Центра регистрации для подписания запросов на выпуск сертификатов.

### Выпуск сертификатов безопасности на основе всех ранее созданных запросов

После того как все запросы на выпуск сертификатов были размещены в домашней директори учетной записи _cpca-srv_ Центра сертификации, можно выпустить целевые сертификаты. Для этого используем плейбук 
[play/21.cproca-certs-issue.yml](vagrant/ansible.ca/play/21.cproca-certs-issue.yml):
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 21 >>> ISSUE OF CERTS | Issue a certificates.
  hosts: cprocaserver
  become: true
  become_user: "{{ cpca_srv_acc }}"
  tasks:
    - name: ISSUE OF CERTS
      ansible.builtin.shell: |
        test -f {{ ansible_user_dir }}/{{ item }}.cer || ./{{ pkicabin }} ca issue-cert --file {{ ansible_user_dir }}/{{ item }}.req --out-file {{ ansible_user_dir }}/{{ item }}.cer --name "{{ caname }}"
      args:
        executable: /bin/bash
        chdir: "{{ pkicabase }}"
      loop:
        - "{{ crt_ca_natsclnt }}"
        - "{{ crt_ca_natssrv }}"
        - "{{ crt_ca_pkicaclnt }}"
        - "{{ crt_ca_caclnt }}"
        - "{{ crt_ra_srv }}"
        - "{{ crt_ra_clnt }}"
        - "{{ crt_ra_webclnt }}"
        - "{{ crt_ra_pkicaclnt }}"
        - "{{ crt_opertls }}"
        - "{{ crt_opersign }}"
```
</details>

Здесь мы выпустили сертификаты на основе запросов, созданных на:
  - :white_check_mark: Центре сертификации:
    - :heavy_check_mark: клиентской части службы очередей _Nats_;
    - :heavy_check_mark: серверной части службы очередей _Nats_;
    - :heavy_check_mark: утилиты _Pkica_ Центра сертификации для подключения к _Nats_;
    - :heavy_check_mark: службы Центра сертификации для подключения к _Nats_;
  - :white_check_mark: Центре регистрации:
    - :heavy_check_mark: веб сервера Центра регистрации;
    - :heavy_check_mark: службы Центра регистрации для подключения к _Nats_;
    - :heavy_check_mark: веб службы Центра регистрации для подключения к _Nats_;
    - :heavy_check_mark: утилиты _pkica_Центра регистрации для подключения к _Nats_;
    - :heavy_check_mark: оператора Центра регистрации для создания защищённого подключения;
    - :heavy_check_mark: оператора Центра регистрации для подписания запросов на выпуск сертификатов.
