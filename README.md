## Установка и первоначальная настройка КриптоПро УЦ 2.0 для Linux (сборка 1.63.0.32) в среде виртуализации 

Пример установки и первоначальной настройки удостоверяющего центра на базе программного обеспечения от _КриптоПро_. В данном примере попытаемся, используя средства виртуализации, 
развернуть УЦ от [КриптоПро](https://cryptopro.ru/products/ca/2.0) для ознакомления с функционалом и тестирования.
> [!IMPORTANT]
> Сразу отмечу, что рассматриваемая здесь установка _КриптоПро УЦ 2.0_ **не подразумевает** использование его в промышленных целях и не является руководством для 
> подобного применения, так как в данной реализации нет соответствия множеству требованиий, содержащихся в эксплуатационной документации и подлежащих 
> исполнению при работе с _СКЗИ_ со стороны действующего законодательства.

Стенд, на котором будет развернут комплекс, представляет собой хост-машину, где в качестве гипервизора установлено ПО виртуализации _KVM_ и среда разработки - _Vagrant_.
Все узлы _Удостоверяющего Центра_ - это _QEMU_-образы виртуальных машин, которые разворачиваются с помощью средств автоматизации и оркестрации _Vagrant_ и _Ansible_.

### Установка и первоначальная настройка виртуальных машин с помощью _Vagrant_

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
