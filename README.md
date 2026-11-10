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

В данном _Vagrantfile_ приведено описание четырёх виртуальных машин:
  - _Debian121_ - сервер Центра сертификации;
  - _Debian122_ - сервер Центра регистрации;
  - _Debian123_ - сервер баз данных;
  - _Debian124_ - рабочая станция оператора, соответственно. 

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

[cprodesktopserver]
cprodesktop ansible_host=192.168.121.14 ansible_port=22 ansible_private_key_file=/home/max/vagrant/.vagrant/machines/Debian124/libvirt/private_key

[caservers:children]
cprocaserver
cproraserver

[cspservers:children]
cprocaserver
cproraserver
cprodesktopserver

[allservers:children]
cprocaserver
cproraserver
cprodbserver
cprodesktopserver
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
Загрузка осуществляется с помощью роли [01.copy-distro](vagrant/ansible.ca/roles/01.copy-distro), плейбук [01.cproca-copy-distr.yml](vagrant/ansible.ca/01.copy-distro.yml)
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 01 >>> COPY DISTRO | Group of servers "caservers".
  hosts: cspservers
  become: false
  roles:
    - 01.copy-distro
```
</details>


### Установка КриптоПро CSP 5.0 R3

Далее потребуется распаковать ранее загруженный дистрибутив КриптоПро CSP, запустить вложенный скрипт __install.sh__ с необходимым набором параметров (уровни кс{1,2} и набор компонентов), 
ввести лицензии, добавить гамму и перезапустить службу cprocsp. 
Все эти действия выполняет роль [02.cprocsp-install](vagrant/ansible.ca/roles/02.cprocsp-install) плейбук [02.cprocsp-install.yml](vagrant/ansible.ca/02.cprocsp-install.yml)
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 02 >>> INSTALL CSP | Extract Archive. Install distributives CryptoPro CSP. Setup Licenses for CSP. Install Gamma.
  hosts: cspservers
  roles:
    - 02.cprocsp-install

```
</details>

Данная роль выполняет следующие шаги:
  - :white_check_mark: создание каталога, в случае его отсутствия, и распаковка дистрибутива в этот каталог;
  - :white_check_mark: запуск скрипта для установки _Криптопро CSP_ с заданными ключами;
  - :white_check_mark: ввод лицензии на использование _Криптопро CSP_;
  - :white_check_mark: копирование ранее созданной гаммы и задание прав доступа к ней;
  - :white_check_mark: перезапуск службы криптопрвайдера.

### Установка PostgreSQL Server
Клиентская и серверная часть Postgresql устанавливается из репозиториев Debian. Для этого используем роли [03.pgsql-client-install](vagrant/ansible.ca/roles/03.pgsql-client-install), 
[03.pgsql-server-install](vagrant/ansible.ca/roles/03.pgsql-server-install)  плейбук [03.pgsql-install.yml](vagrant/ansible.ca/03.pgsql-install.yml)
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 03 >>> APT. INSTALL POSTGRESQL | Install Client.
  hosts: caservers
  become: true
  roles:
    - 03.pgsql-client-install

- name: <<< PLAYBOOK 03 >>> APT. INSTALL POSTGRESQL | Install Server.
  hosts: cprodbserver
  become: true
  roles:
    - 03.pgsql-server-install
```

</details>

Здесь мы для серверов __cproca__ и __cprora__ (группа хостов __caservers__ в файле __staging/hosts__) установили пакеты:
  - :white_check_mark: acl
  - :white_check_mark: postgresql-client
  - :white_check_mark: python3-cryptography
  - :white_check_mark: устаревший пакет __libssl1__ - он требуется только на современных дистрибутивах Linux.

Для сервера __cprodbserver__ же, были установлены:
  - :white_check_mark: acl
  - :white_check_mark: postgresql
  - :white_check_mark: python3-psycopg2


### Настройка PostgreSQL Server. Предоставление доступа для внешних подключений
Откроем доступ для подключения к базам данных с удалённых хостов cproca-test и cprora-test. Роль [04.pgsql-server-config](vagrant/ansible.ca/roles/04.pgsql-server-config), 
плейбук [04.pgsql-config.yml](vagrant/ansible.ca/04.pgsql-config.yml)
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 04 >>> CONFIGURE POSTGRESQL | Configure Server.
  hosts: cprodbserver
  become: true
  roles:
    - 04.pgsql-server-config
```

</details>

Данный плейбук выполняет редактирование файла __pg_hba__, а именно открывает следующие доступы ко всем базам данных:
  - :white_check_mark: с хоста __cproca__ для пользователя __cpca\_dbadmin__
  - :white_check_mark: с хоста __cprora__ для пользователя __cpra\_dbadmin__

### Настройка PostgreSQL Server. Создание комплекта баз данных и ролей, настройка привилегий
Создание баз данных и ролей, а так-же, предоставление привилегий для этих ролей на созданных базах. Роль [05.pgsql-create-dbs-roles](vagrant/ansible.ca/roles/05.pgsql-create-dbs-roles) плейбук [05.pgsql-create-dbs-roles.yml](vagrant/ansible.ca/05.pgsql-create-dbs-roles.yml)
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 05 >>> POSTGRESQL | Configure Server.
  hosts: cprodbserver
  become: true
  become_user: postgres
  roles:
    - 05.pgsql-create-dbs-roles
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
> А в данном случае были созданы роли и одноименные базы данных, так как они требуются для подключения этой утилиты к серверу баз данных 
> (Утилита _pkica_ при подключении к Postgresql использует формат логина _username@username_). 
> Если вместо служебных ролей __cpca_dbadmin__ и ___cpra_dbadmin__ в настройках подключения использовать встроенную роль _Postgres_, то данный шаг выполнять не требуется. 

### Центр сертификации и Центр регистрации. Создание группы безопасности и служебных пользователей. Настройка файлов аутентификации на сервере баз данных
Для установки CRL в хранилище __LocalMachine\CA__ на серверах Центра Сертификации и Центра Регистрации создается группа __crl-writers__. В эту группу добавляется служебный пользователь, с правами которого работают службы ЦС и ЦР.
Так же, для служебных учетных записей настраивается подключение к базам данных. Плейбук [06.system-create-groups-users.yml](vagrant/ansible.ca/06.system-create-groups-users.yml):
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 06 >>> CPROCA AND CPRORA Servers | Create group crl-writers and set ACLs. Create user with admins privileges
  hosts: caservers
  become: true
  roles:
    - 06.system-group-user-create

- name: <<< PLAYBOOK 06 >>> CPROCA | Create CryptoPro Ca service login
  hosts: cprocaserver
  become: true
  roles:
    - 06.system-caservice-user-create

- name: <<< PLAYBOOK 06 >>> CPRORA | Create user CryptoPro Ra service login
  hosts: cproraserver
  become: true
  roles:
    - 06.system-raservice-user-create
```
</details>

### Центр сертификации и Центр регистрации. Распаковка дистрибутива, копирование файлов в рабочую директорию. Установка разрешений
Установка службы Центра Сертификации, как и Центра Регистрации - это простое копирование содержимого архива ca-linux-x64-x.xx.x.xx.zip в каталог /opt/cpca.

После копирования необходимо дать сервисным учетным записям __cpca-srv__ и __cpra-srv__ права на запуск соответствующих исполняемых файлов - **CryptoPro.Ca.Service**, **CryptoPro.Ra.Service** и **CryptoPro.Ra.Web**. 
Плейбук [07.cproca-extract-distro-set-permissions.yml](vagrant/ansible.ca/07.cproca-extract-distro-set-permissions.yml):
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 07 >>> CPROCA and CPRORA | Install package CPCA.
  hosts: caservers
  become: true
  roles:
    - 07.cproca-extract-distro

- name: <<< PLAYBOOK 07 >>> CPROCA | Set permissions and ACL privileges for CryptoPro.Ca.Service and Pkica util
  hosts: cprocaserver
  become: true
  roles:
    - 07.caservice-set-perms

- name: <<< PLAYBOOK 07 >>> CPRORA | Set permissions and ACL privileges for CryptoPro.Ra.Service and Pkica util. Install Nginx
  hosts: cproraserver
  become: true
  roles:
    - 07.raservice-set-perms

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

Плейбук [08.config-distros-set-parameters.yml](vagrant/ansible.ca/08.config-distros-set-parameters.yml):
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 08.1 >>> CPROCA AND CPRORA | Set parameters for PKICA appsettings.json
  hosts: caservers
  become: true
  roles:
    - 08.config-distros-set-parameters-pkica

- name: <<< PLAYBOOK 08.2 >>> CPROCA | Set parameters for CA appsettings.json
  hosts: cprocaserver
  become: true
  roles:
    - 08.config-distros-set-parameters-ca

- name: <<< PLAYBOOK 08.3 >>> CPRORA | Set parameters for RA and RAWEB appsettings.json
  hosts: cproraserver
  become: true
  roles:
    - 08.config-distros-set-parameters-ra
    - 08.config-distros-set-parameters-raweb

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

Далее, для служб: 
  - :white_check_mark: __CryptoPro CA__ на сервере Центра сертификации:
    - :heavy_check_mark: отредактировали строку подключения к базе данных Центра сертификации; 
    - :heavy_check_mark: отключили шифрование для подключения к службе __Nats__;
    - :heavy_check_mark: отключили шифрование для подключения к службе __Stan__;
    - :heavy_check_mark: указали в качестве адреса для подключения к службе __Nats__ собственное имя хоста вместо __localhost__;
    - :heavy_check_mark: указали в качестве адреса для подключения к службе __Stan__ собственное имя хоста вместо __localhost__;
    - :heavy_check_mark: ввели серийный номер и название компании;
  - :white_check_mark: __nats-streaming-server__:
    - :heavy_check_mark: изменили адрес, на котором работает служба.

Для службы __CryptoPro.Ra.Service__ на сервере Центра регистрации: 
  - :white_check_mark: отредактировали строку подключения к базе данных Центра регистрации;
  - :white_check_mark: отключили шифрование для подключения к службе __Nats__;
  - :white_check_mark: отключили шифрование для подключения к службе __Stan__;
  - :white_check_mark: указали в качестве адреса для подключения к службе __Nats__ имя хоста центра сертификации вместо __localhost__;
  - :white_check_mark: указали в качестве адреса для подключения к службе __Stan__ имя хоста центра сертификации вместо __localhost__;
  - :white_check_mark: включили публикацию списков отзыва;
  - :white_check_mark: включили публикацию сертификтов ЦС;
  - :white_check_mark: ввели серийный номер и название компании.

И для службы __CryptoPro.Ra.Web.Service__ на сервере Центра регистрации: 
  - :white_check_mark: отредактировали строку подключения к базе данных Центра регистрации;
  - :white_check_mark: отключили шифрование для подключения к службе __Nats__;
  - :white_check_mark: отключили шифрование для подключения к службе __Stan__;
  - :white_check_mark: изменили адрес для подключения к службе с _localhost_ на собственное имя хоста;
  - :white_check_mark: указали в качестве адреса для подключения к службе __Nats__ имя хоста центра сертификации вместо __localhost__;
  - :white_check_mark: указали в качестве адреса для подключения к службе __Stan__ имя хоста центра сертификации вместо __localhost__;
  - :white_check_mark: ввели серийный номер и название компании.

#### Способ 2. Использование ansible-модуля lineinfile

В предыдущем плейбуке все изменения производились с помощью модуля __shell__ и текстового процессора __sed__ - т.е. использовался т.н. _bashsible_. 

Здесь же, для получения того же результата, к которому приводит работа плейбука [08.config-distros-set-parameters.yml](vagrant/ansible.ca/08.config-distros-set-parameters.yml), 
мы будем пользоваться модулем __lineinfile__.  Для этого используем следующий плейбук [09.config-distros-set-parameters.yml](vagrant/ansible.ca/09.config-distros-set-parameters.yml):

<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 09.1 >>> CPROCA AND CPRORA | Config "ConnectionString" parameter for Pkica appsettings.json
  hosts: caservers
  become: true
  roles:
    - 09.config-distros-set-parameters-pkica

- name: <<< PLAYBOOK 09.2 >>> CPROCA | Set parameters for CA appsettings.json
  hosts: cprocaserver
  become: true
  roles:
  - 09.config-distros-set-parameters-ca

- name: <<< PLAYBOOK 09.3 >>> CPRORA | Set parameters for RA and RAWEB appsettings.json
  hosts: cproraserver
  become: true
  roles:
  - 09.config-distros-set-parameters-ra
  - 09.config-distros-set-parameters-raweb

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

Далее, для служб:
  - :white_check_mark: __CryptoPro CA__ на сервере Центра сертификации:
    - :heavy_check_mark: отредактировали строку подключения к базе данных Центра сертификации;
    - :heavy_check_mark: отключили шифрование для подключения к службе __Nats__;
    - :heavy_check_mark: указали в качестве адреса для подключения к службе __Nats__ собственное имя хоста вместо __localhost__;
    - :heavy_check_mark: отключили шифрование для подключения к службе __Stan__;
    - :heavy_check_mark: указали в качестве адреса для подключения к службе __Stan__ собственное имя хоста вместо __localhost__;
    - :heavy_check_mark: ввели серийный номер и название компании;
  - :white_check_mark: __nats-streaming-server__ - Служба очередей NATS Streaming с поддержкой ГОСТ TLS:
    - :heavy_check_mark: изменили адрес, на котором работает эта служба.

Для службы __CryptoPro.Ra.Service__ на сервере Центра регистрации:
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

И для службы __CryptoPro.Ra.Web.Service__ на сервере Центра регистрации:
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
  hosts: cprocaserver
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
  hosts: cproraserver
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
  hosts: cproraserver
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
  hosts: cprocaserver
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
  hosts: cprocaserver
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
  hosts: cprocaserver
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
  hosts: cprocaserver
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
  hosts: cproraserver
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
  hosts: cprocaserver
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

### Выпуск сертификатов на основе всех ранее созданных запросов

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
  - :white_check_mark: Центре сертификации для:
    - :heavy_check_mark: клиентской части службы очередей _Nats_;
    - :heavy_check_mark: серверной части службы очередей _Nats_;
    - :heavy_check_mark: утилиты _pkica_ Центра сертификации для подключения к _Nats_;
    - :heavy_check_mark: службы Центра сертификации для подключения к _Nats_;
  - :white_check_mark: Центре регистрации для:
    - :heavy_check_mark: веб сервера Центра регистрации;
    - :heavy_check_mark: службы Центра регистрации для подключения к _Nats_;
    - :heavy_check_mark: веб службы Центра регистрации для подключения к _Nats_;
    - :heavy_check_mark: утилиты _pkica_ Центра регистрации для подключения к _Nats_;
    - :heavy_check_mark: оператора Центра регистрации для создания защищённого подключения;
    - :heavy_check_mark: оператора Центра регистрации для подписания запросов на выпуск сертификатов.

Теперь сертификаты, котрые были выпущены для _Центра регистрации_, необходио будет скопировать обратно на сервер ЦР.

### Копирование выпущенных сертификатов с ЦС на ЦР

Для передачи сертификатов с _Центра Сертификации_ на _Центр Регистрации_ задействуется управляющая машина. Таким образом, файлы сертификатов сначала скачиваются 
на хост управления с _Центра Сертификации_, затем загружаются с него на _Центр Регистрации_.
Для этого задействуем дла плейбука, [play/22.cproca-certs-fetch.yml](vagrant/ansible.ca/play/22.cproca-certs-fetch.yml):
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 22 >>> FETCH CERTS | Download certificates from CA.
  hosts: cprocaserver
  become: true
  become_user: "{{ cpca_srv_acc }}"
  tasks:
    - name: FETCH.
      ansible.builtin.fetch:
        src: "{{ ansible_user_dir }}/{{ item }}.cer"
        dest: "{{ localdir }}/share/"
        flat: 'true'
      loop:
        - "{{ crt_ra_srv }}"
        - "{{ crt_ra_clnt }}"
        - "{{ crt_ra_webclnt }}"
        - "{{ crt_ra_pkicaclnt }}"
        - "{{ crt_opertls }}"
        - "{{ crt_opersign }}"
        - "{{ cacert }}"
```
</details>

и [play/23.cprora-certs-up.yml](vagrant/ansible.ca/play/23.cprora-certs-up.yml):
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 23 >>> UP CERIFICATES | Upload certificates to RA.
  hosts: cproraserver
  become: true
  become_user: "{{ cpra_srv_acc }}"
  tasks:
    - name: UP.
      ansible.builtin.copy:
        src: "{{ localdir }}/share/{{ item }}.cer"
        dest: "{{ ansible_user_dir }}/{{ item }}.cer"
      loop:
        - "{{ crt_ra_srv }}"
        - "{{ crt_ra_clnt }}"
        - "{{ crt_ra_webclnt }}"
        - "{{ crt_ra_pkicaclnt }}"
        - "{{ crt_opertls }}"
        - "{{ crt_opersign }}"
        - "{{ cacert }}"
```
</details>

### Установка сертификатов на серверах Центра Сертификации и Центра Регистрации

Установка сертификатов в хранилище с привязкой к закрытому ключу для учетных записей, с правами которых работают службы. 
На Центре сертификации - плейбук [play/24.cproca-certs-install.yml](vagrant/ansible.ca/play/24.cproca-certs-install.yml):
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 24 >>> INSTALL | Install certificates on a Certifications Authority.
  hosts: cprocaserver
  become: true
  become_user: "{{ cpca_srv_acc }}"
  tasks:
    - name: INSTALL
      ansible.builtin.shell: |
        ./{{ cryptcpbin }} -instcert -from_request {{ ansible_user_dir }}/{{ item }}.cer
      args:
        executable: /bin/bash
        chdir: "{{ cspbindir }}"
      loop:
        - "{{ crt_ca_natsclnt }}"
        - "{{ crt_ca_natssrv }}"
        - "{{ crt_ca_pkicaclnt }}"
        - "{{ crt_ca_caclnt }}"
```
</details>

На Центре регистрации - плейбук [play/25.cprora-certs-install.yml](vagrant/ansible.ca/play/25.cprora-certs-install.yml):
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 25 >>> INSTALL | Install certificates on a Registrations Authority.
  hosts: cproraserver
  become: true
  become_user: "{{ cpra_srv_acc }}"
  tasks:
    - name: CPRORA. Install Cerificates to uMy
      ansible.builtin.shell: |
        ./{{ cryptcpbin }} -instcert -from_request {{ ansible_user_dir }}/{{ item }}.cer
      args:
        executable: /bin/bash
        chdir: "{{ cspbindir }}"
      loop:
        - "{{ crt_ra_srv }}"
        - "{{ crt_ra_clnt }}"
        - "{{ crt_ra_webclnt }}"
        - "{{ crt_ra_pkicaclnt }}"
        - "{{ crt_opertls }}"
        - "{{ crt_opersign }}"

    - name: CPRORA. Export Cerificates to pfx
      ansible.builtin.shell: |
        export PATH=$PATH:{{ cspbindir }}
        id=$(certmgr -list -file {{ ansible_user_dir }}/{{ item }}.cer | grep "SHA1" | awk '{print $4}')
        certmgr -export -pfx -pin 11111111 -thumbprint $id -dest {{ ansible_user_dir }}/{{ item }}.pfx
      args:
        executable: /bin/bash
      loop:
        - "{{ crt_ra_srv }}"
        - "{{ crt_opertls }}"
        - "{{ crt_opersign }}"
      tags: export

- name: <<< PLAYBOOK 25 >>> INSTALL | Install rootca certificate on a Registrations Authority.
  hosts: cproraserver
  become: true
  tasks:
    - name: CPRORA. Install Cerificates to mRoot
      ansible.builtin.shell: |
        # export PATH=$PATH:"{{ cspbindir }}"
        yes o | ./certmgr -install -store mRoot -file /home/{{ cpra_srv_acc }}/{{ cacert }}.cer
      args:
        executable: /bin/bash
        chdir: "{{ cspbindir }}"

```
</details>

Здесь мы установили сертификаты на:
  - :white_check_mark: Центре сертификации:
    - :heavy_check_mark: клиентской части службы очередей _Nats_;
    - :heavy_check_mark: серверной части службы очередей _Nats_;
    - :heavy_check_mark: утилиты _pkica_ Центра сертификации для подключения к _Nats_;
    - :heavy_check_mark: службы Центра сертификации для подключения к _Nats_;
  - :white_check_mark: Центре регистрации:
    - :heavy_check_mark: веб сервера Центра регистрации;
    - :heavy_check_mark: службы Центра регистрации для подключения к _Nats_;
    - :heavy_check_mark: веб службы Центра регистрации для подключения к _Nats_;
    - :heavy_check_mark: утилиты _pkica_ Центра регистрации для подключения к _Nats_;
    - :heavy_check_mark: оператора Центра регистрации для создания защищённого подключения;
    - :heavy_check_mark: оператора Центра регистрации для подписания запросов на выпуск сертификатов.
    - :heavy_check_mark: корневой сертификат Центра сертификации

Также, на Центре регистрации, мы экспортировали сертификаты в формате pfx:
  - :white_check_mark: веб сервера Центра регистрации - для установки в локальное хранилище пользователя _nginx_;
  - :white_check_mark: оператора Центра регистрации (для создания защищённого подключения) для передачи и последующей установки на рабочую станцию оператора;
  - :white_check_mark: оператора Центра регистрации (для подписания запросов на выпуск сертификатов) для передачи и последующей установки на рабочую станцию оператора;

### Копирование и установка сертификатов оператора на его рабочую станцию

После того, как сертификаты оператора были экспортированиы из хранилища в файлы _pfx_, их необходимо скопировать на рабочую станцию и там установить в локальное хранилище учетной записи оператора,
в данном случае это пользователь _Vagrant_.

Плейбук [play/26.cprora-pfxs-fetch.yml](vagrant/ansible.ca/play/26.cprora-pfxs-fetch.yml) - копирует файлы _pfx_ с _Центра регистрации_ на управляющую машину в каталог _share_:

<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 26 >>> FETCH PFXs | Download keys from RA.
  hosts: cproraserver
  become: true
  become_user: "{{ cpra_srv_acc }}"
  tasks:
    - name: FETCH.
      ansible.builtin.fetch:
        src: "{{ ansible_user_dir }}/{{ item }}.pfx"
        dest: "{{ localdir }}/share/"
        flat: 'true'
      loop:
        - "{{ crt_opertls }}"
        - "{{ crt_opersign }}"
```
</details>

Плейбук [play/27.cprodesktop-pfx-up.yml](vagrant/ansible.ca/play/27.cprodesktop-pfx-up.yml) - копирует файлы _pfx_ и корневой сертификат Центра сертификации с каталога _share_ управляющей машины на станцию оператора 
в домашнюю директорию пользователя _Vagrant_:
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 27 >>> UP PFXs | Upload keys to Desktop.
  hosts: cprodesktopserver
  become: false
  tasks:
    - name: UP.
      ansible.builtin.copy:
        src: "{{ localdir }}/share/{{ item }}"
        dest: "{{ ansible_user_dir }}/{{ item }}"
      loop:
        - "{{ cacert }}.cer"
        - "{{ crt_opertls }}.pfx"
        - "{{ crt_opersign }}.pfx"
```
</details>

Плейбук [play/28.cprodesktop-pfx-install.yml](vagrant/ansible.ca/play/28.cprodesktop-pfx-install.yml) - устанавливает сертификаты с привязкой к закрытому ключу в хранилище пользователя __uMy__, 
а также корневой сертифкат в хранилище __uRoot__:
<details>
<summary>Клик, чтобы показать код :arrow_down_small:</summary>

```
---
- name: <<< PLAYBOOK 28 >>> CPRODesktop | Install a Certificate for Operator
  hosts: cprodesktopserver
  become: false
  tasks:
    - name: CryptoPro Desktop. Install a Certificates for Operator
      ansible.builtin.shell: |
        ./certmgr -install -pfx -store uMy -file "{{ ansible_user_dir }}/{{ item }}" -newpin '' -pin 11111111
      args:
        executable: /bin/bash
        chdir: "{{ cspbindir }}"
      loop:
        - "{{ crt_opertls }}.pfx"
        - "{{ crt_opersign }}.pfx"

- name: <<< PLAYBOOK 28 >>> Install rootca certificate on a Desktop.
  hosts: cprodesktopserver
  become: false
  tasks:
    - name: CPRORA. Install Cerificates to uRoot
      ansible.builtin.shell: |
        yes o | ./certmgr -install -store uRoot -file "{{ ansible_user_dir }}/{{ cacert }}.cer"
      args:
        executable: /bin/bash
        chdir: "{{ cspbindir }}"

```
</details>
