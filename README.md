### Установка и первоначальная настройка КриптоПро УЦ 2.0 для Linux (сборка 1.63.0.32) в среде виртуализации 

Пример установки и первоначальной настройки удостоверяющего центра на базе программного обеспечения от _КриптоПро_. В данном примере попытаемся, используя средства виртуализации, 
развернуть УЦ от [КриптоПро](https://cryptopro.ru/products/ca/2.0) для ознакомления с функционалом и тестирования.
> [!IMPORTANT]
> Сразу отмечу, что рассматриваемая здесь установка _КриптоПро УЦ 2.0_ **не подразумевает** использование его в промышленных целях и не является руководством для 
> подобного применения, так как в данной реализации не происходит соответствие множеству требованиий, содержащихся в эксплуатационной документации и подлежащих 
> исполнению при работе с _СКЗИ_ со стороны действующего законодательства.

Стенд, на котором будет развернут комплекс, представляет собой хост-машину, где в качестве гипервизора установлено ПО виртуализации _KVM_ и среда разработки - _Vagrant_.
Все узлы _Удостоверяющего Центра_ - это _QEMU_ образы виртуальных машин, которые разворачиваются с помощью средств автоматизации и оркестрации _Vagrant_ и _Ansible_.

#### Установка и первоначальная настройка виртуальных машин с помощью _Vagrant_

> [!NOTE]
> Создание _Vagrant_-образов виртуальных машин рассматривается в данной [статье](https://github.com/spanishairman/vagrant).
 
Описание виртуальных машин в _Vagrantfile_ выглядит так:
<details>
<summary>Vagrantfile code</summary>

```
# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure("2") do |config|
  config.vm.define "Astra17-cproca1" do |cpca1server|
  cpca1server.vm.box = "/home/max/vagrant/images/astralinux17g"
  cpca1server.vm.network :private_network,
       :type => 'ip',
       :libvirt__forward_mode => 'veryisolated',
       :libvirt__dhcp_enabled => false,
       :ip => '192.168.1.3',
       :libvirt__netmask => '255.255.255.248',
       :libvirt__network_name => 'vagrant-libvirt-inet1',
       :libvirt__always_destroy => false
  cpca1server.vm.provider "libvirt" do |lvirt|
      lvirt.memory = "1024"
      lvirt.cpus = "1"
      lvirt.title = "Astra17-cproca1Server"
      lvirt.description = "Виртуальная машина на базе дистрибутива Debian Linux. cproca1Server"
      lvirt.management_network_name = "vagrant-libvirt-mgmt"
      lvirt.management_network_address = "192.168.121.0/24"
      lvirt.management_network_keep = "true"
      lvirt.management_network_mac = "52:54:00:27:28:85"
  end
  cpca1server.vm.provision "file", source: "cprokey/ca-linux-x64-1.63.0.32.zip", destination: "~/ca-linux-x64-1.63.0.32.zip"
  cpca1server.vm.provision "file", source: "cprokey/linux-amd64_deb.tgz", destination: "~/linux-amd64_deb.tgz"
  cpca1server.vm.provision "file", source: "cprokey/pgpass", destination: "~/.pgpass"
  cpca1server.vm.provision "file", source: "cprokey/cryptopro.ca.service", destination: "~/cryptopro.ca.service"
  cpca1server.vm.provision "file", source: "cprokey/cryptopro.ra.service", destination: "~/cryptopro.ra.service"
  cpca1server.vm.provision "file", source: "cprokey/nats-streaming-server.service", destination: "~/nats-streaming-server.service"
  cpca1server.vm.provision "file", source: "cprokey/Crypto\ Pro", destination: "~/"
  cpca1server.vm.provision "shell", inline: <<-SHELL
      brd='*************************************************************'
      chown vagrant: /home/vagrant/.pgpass
      chmod 600 /home/vagrant/.pgpass
      cp -p /home/vagrant/.pgpass ~/.pgpass
      chown root: ~/.pgpass
      echo "$brd"
      echo 'Set Hostname'
      hostnamectl set-hostname cpca1server
      echo "$brd"
      sed -i 's/astra17/cpca1server/' /etc/hosts
      sed -i 's/astra17/cpca1server/' /etc/hosts
      echo '192.168.1.1 cpk1server.local cpk1server' >> /etc/hosts
      echo '192.168.1.2 cpk2server.local cpk2server' >> /etc/hosts
      echo '192.168.1.3 cpca1server.local cpca1server' >> /etc/hosts
      echo '192.168.1.4 cpca2server.local cpca2server' >> /etc/hosts
      echo '192.168.1.5 psql1server.local psql1server' >> /etc/hosts
      echo '192.168.1.6 psql2server.local psql2server' >> /etc/hosts
      echo "$brd"
      echo 'Изменим ttl для работы через раздающий телефон'
      echo "$brd"
      sysctl -w net.ipv4.ip_default_ttl=66
      echo "$brd"
      echo 'Если ранее не были установлены, то установим необходимые  пакеты'
      echo "$brd"
      mount -o loop /home/vagrant/installation-1.7.5.9-16.10.23_16.58.iso /media/localiso/
      export DEBIAN_FRONTEND=noninteractive
      apt update
      # Политика по умолчанию для цепочки INPUT - DROP
      iptables -P INPUT DROP
      # Политика по умолчанию для цепочки OUTPUT - DROP
      iptables -P OUTPUT DROP
      # Политика по умолчанию для цепочки FORWARD - DROP
      iptables -P FORWARD DROP
      # Базовый набор правил: разрешаем локалхост, запрещаем доступ к адресу сети обратной петли не от локалхоста,
      # разрешаем входящие пакеты со статусом установленного сонединения.
      iptables -A INPUT -i lo -j ACCEPT
      iptables -A INPUT ! -i lo -d 127.0.0.0/8 -j REJECT
      iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
      # Открываем исходящие
      iptables -A OUTPUT -j ACCEPT
      # Разрешим входящие с хоста управления.
      iptables -A INPUT -s 192.168.121.1 -m tcp -p tcp --dport 22 -j ACCEPT
      # Разрешим входящие для изолированной сети
      iptables -A INPUT -s 192.168.1.0/29 -m tcp -p tcp -j ACCEPT
      # Откроем ICMP ping
      iptables -A INPUT -p icmp -m icmp --icmp-type 8 -j ACCEPT
      # ip route add 192.168.1.8/29 via 192.168.1.1
      # ip route save > /etc/my-routes
      # echo 'up ip route restore < /etc/my-routes' >> /etc/network/interfaces
      SHELL
  end
  config.vm.define "Astra17-psql1" do |psql1server|
  psql1server.vm.box = "/home/max/vagrant/images/astralinux17g"
  psql1server.vm.network :private_network,
       :type => 'ip',
       :libvirt__forward_mode => 'veryisolated',
       :libvirt__dhcp_enabled => false,
       :ip => '192.168.1.5',
       :libvirt__netmask => '255.255.255.248',
       :libvirt__network_name => 'vagrant-libvirt-inet1',
       :libvirt__always_destroy => false
  psql1server.vm.provider "libvirt" do |lvirt|
      lvirt.memory = "1024"
      lvirt.cpus = "1"
      lvirt.title = "Astra17-psql1Server"
      lvirt.description = "Виртуальная машина на базе дистрибутива Debian Linux. psql1Server"
      lvirt.management_network_name = "vagrant-libvirt-mgmt"
      lvirt.management_network_address = "192.168.121.0/24"
      lvirt.management_network_keep = "true"
      lvirt.management_network_mac = "52:54:00:27:28:87"
  end
  psql1server.vm.provision "shell", inline: <<-SHELL
      brd='*************************************************************'
      echo "$brd"
      echo 'Set Hostname'
      hostnamectl set-hostname psql1server
      echo "$brd"
      sed -i 's/astra17/psql1server/' /etc/hosts
      sed -i 's/astra17/psql1server/' /etc/hosts
      echo '192.168.1.1 cpk1server.local cpk1server' >> /etc/hosts
      echo '192.168.1.2 cpk2server.local cpk2server' >> /etc/hosts
      echo '192.168.1.3 cpca1server.local cpca1server' >> /etc/hosts
      echo '192.168.1.4 cpca2server.local cpca2server' >> /etc/hosts
      echo '192.168.1.5 psql1server.local psql1server' >> /etc/hosts
      echo '192.168.1.6 psql2server.local psql2server' >> /etc/hosts
      echo "$brd"
      echo 'Изменим ttl для работы через раздающий телефон'
      echo "$brd"
      sysctl -w net.ipv4.ip_default_ttl=66
      echo "$brd"
      echo 'Если ранее не были установлены, то установим необходимые  пакеты'
      echo "$brd"
      mount -o loop /home/vagrant/installation-1.7.5.9-16.10.23_16.58.iso /media/localiso/
      export DEBIAN_FRONTEND=noninteractive
      apt update
      # Политика по умолчанию для цепочки INPUT - DROP
      iptables -P INPUT DROP
      # Политика по умолчанию для цепочки OUTPUT - DROP
      iptables -P OUTPUT DROP
      # Политика по умолчанию для цепочки FORWARD - DROP
      iptables -P FORWARD DROP
      # Базовый набор правил: разрешаем локалхост, запрещаем доступ к адресу сети обратной петли не от локалхоста,
      # разрешаем входящие пакеты со статусом установленного сонединения.
      iptables -A INPUT -i lo -j ACCEPT
      iptables -A INPUT ! -i lo -d 127.0.0.0/8 -j REJECT
      iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
      # Открываем исходящие
      iptables -A OUTPUT -j ACCEPT
      # Разрешим входящие с хоста управления.
      iptables -A INPUT -s 192.168.121.1 -m tcp -p tcp --dport 22 -j ACCEPT
      # Разрешим входящие для изолированной сети
      iptables -A INPUT -s 192.168.1.0/29 -m tcp -p tcp -j ACCEPT
      # Откроем ICMP ping
      iptables -A INPUT -p icmp -m icmp --icmp-type 8 -j ACCEPT
      # ip route add 192.168.1.8/29 via 192.168.1.1
      # ip route save > /etc/my-routes
      # echo 'up ip route restore < /etc/my-routes' >> /etc/network/interfaces
      SHELL
  end
```
</details>

В данном _Vagrantfile_ приведено описание двух виртуальных машин: _cpca1server_ и _psql1server_ - сервер _Центра сертификации/Центра регистрации_ и сервер баз данных соответственно. 
В качестве _СУБД_ используется _PostgreSQL_, установленная из репозиториев операционной системы. 
В соответствии с прилагаемым к _КриптоПро УЦ_ формуляром, будем разворачивать наш УЦ в ОС специального назначения - _Astra linux 1.7.5_.

> [!NOTE]
> Если быть точным, в формуляре к _"КриптоПро УЦ 2.0 1.63.0.32"_ содержится информация о функционировании серверов _Центра сертификации_ и _Центра регистрации_
> в операционной системе специального назначения [«Astra Linux Special Edition» (РУСБ.10015-16)](https://wiki.astralinux.ru/pages/viewpage.action?pageId=331363095) 
> или, что идентично - ["Astra Linux Special Edition РУСБ.10015-01 (очередное обновление 1.6)"](https://wiki.astralinux.ru/pages/viewpage.action?pageId=37290451).

Рассмотрим подробнее блоки "file" виртуальной машины __cpca1server__:
```
  cpca1server.vm.provision "file", source: "cprokey/ca-linux-x64-1.63.0.32.zip", destination: "~/ca-linux-x64-1.63.0.32.zip"
  cpca1server.vm.provision "file", source: "cprokey/linux-amd64_deb.tgz", destination: "~/linux-amd64_deb.tgz"
  cpca1server.vm.provision "file", source: "cprokey/pgpass", destination: "~/.pgpass"
  cpca1server.vm.provision "file", source: "cprokey/cryptopro.ca.service", destination: "~/cryptopro.ca.service"
  cpca1server.vm.provision "file", source: "cprokey/cryptopro.ra.service", destination: "~/cryptopro.ra.service"
  cpca1server.vm.provision "file", source: "cprokey/nats-streaming-server.service", destination: "~/nats-streaming-server.service"
  cpca1server.vm.provision "file", source: "cprokey/Crypto\ Pro", destination: "~/"
```
В данном примере мы загрузили на виртуальную машину __cpca1server__, выступающую в роли УЦ, следующие файлы: 
  - **ca-linux-x64-1.63.0.32.zip** - _"КриптоПро УЦ 2.0 для Linux (сборка 1.63.0.32)"_; 
  - **linux-amd64\_deb.tgz** - _КриптоПро CSP_ версии 5.0.13000-7_;
  - **pgpass** - Файл паролей для подключения к базам данных _PostgreSQL_;
  - **cryptopro.ca.service**, **cryptopro.ra.service**, **nats-streaming-server.service** - Файлы конфигурации служб для демона _SystemD_;
  - **Crypto Pro** - Каталог с ранее сформированной _гаммой_.

Далее в блоке настроек _SHELL_ мы редактируем файл _/etc/hosts_ и задаём базовые правила и политики для _iptables_.

Все создаваемые виртуальные машины имеют два сетевых интерфейса: 
  - _vagrant-libvirt-inet1_ - изолированная сеть, без выхода в интернет и доступа к каким-либо хостам, включая хост виртуализации;
  - _vagrant-libvirt-mgmt_ - сеть управления, с её помощью выполняется управление виртуальными машинами, а также эта сеть позволяет виртуальным машинам выходить в интернет. 

#### Создание пользователей для работы с базами данных _PostgreSQL_, установка и настройка _psqlserver_ на сервере __psql1server__

В _Astra Linux_ пользователь _СУБД_ должен быть также пользователем  операционной системы и иметь права на чтение _атрибутов мандатного разграничения доступа_ 
(_Mandatory Access Control_). Помимо этого пользователь _postgres_, с правами которого работает база данных _PostgreSQL_, должен иметь доступ к базе данных с _MAC_.

С помощью следующего плейбука мы выполним:
  - добавление системных пользователей: задачи __"PostgreSQL. Add the user 'cpca' with a bash shell"__ и __"PostgreSQL. Add the user 'cpra' with a bash shell"__;
  - установку ПО: __"APT. Update the repository cache and install packages "postgresql", "python3-psycopg2", "acl" to latest version"__;
  - предоставим доступы для удаленных пользователей к базам данных _ЦС_ и _ЦР_. Задачи:
    - __Config. Edit pg\_hba configuration file. Add cpca1server access. Открываем доступ с первой ноды сервера CA к базе cpca для пользователя cpca__;
    - __Config. Edit pg\_hba configuration file. Add cpca2server access. Открываем доступ со второй ноды сервера CA к базе cpca для пользователя cpca__;
    - __Config. Edit pg\_hba configuration file. Add cpca1server access. Открываем доступ с первой ноды сервера CA к базе cpra для пользователя cpra__;
    - _Config. Edit pg\_hba configuration file. Add cpca2server access. Открываем доступ со второй ноды сервера CA к базе cpra для пользователя cpra__;
  - отредактируем главный конфигурационный файл _PostgreSQL_: __"Config. Bash. Edit postgresql.conf for enable all interfaces. Разрешаем входящие подключения к порту 5432 на всех интерфейсах__"
  - настроим Mandatory Access Control: __"Config. Give the postgres user rights to read the mandatory access control database"__;
  - создадим базы данных и роли, предостави права для созданных ролей:
    - __Create "cpca" user, and grant access to bases create__;
    - __Create a new database with name "cpca"__;
    - __Connect to "cpca" database, grant privileges on "cpca" database objects (database) for "cpca" role__;
    - __Connect to "cpca" database, grant privileges on "cpca" database objects (schema) for "cpca" role__;
    - __Create "cpra" user, and grant access to bases create__;
    - __Create a new database with name "cpra"__;
    - __Connect to "cpra" database, grant privileges on "cpra" database objects (database) for "cpra" role__;
    - __Connect to cpra database, grant privileges on "cpra" database objects (schema) for "cpra" role__.

> [!NOTE]
> По списку задач можно заметить, что мы открываем доступ к базам данных для удаленных пользователей с первой и второй ноды сервера _CA_. 
> Здесь мы предусматриваем дальнейшую настройку кластера высокой доступности _Центра Сертификации_, в случае необходимости его создания. 

Код плейбука выглядит так:
<details>
<summary>Ansible code</summary>

```
---
- name: PostgreSQL | Group of servers "psqlserver". Create users. Install packages "postgresql", "acl" on the "psqlserver" server group
  hosts: psqlserver
  become: true
  tasks:
    - name: PostgreSQL. Add the user 'cpca' with a bash shell
      ansible.builtin.user:
        name: cpca
        password: '!'
        comment: CryptoPro CA
        shell: /bin/bash
        create_home: yes
    - name: PostgreSQL. Add the user 'cpra' with a bash shell
      ansible.builtin.user:
        name: cpra
        password: '!'
        comment: CryptoPro RA
        shell: /bin/bash
        create_home: yes
    - name: APT. Update the repository cache and install packages "postgresql", "python3-psycopg2", "acl" to latest version
      ansible.builtin.apt:
        name: postgresql,python-psycopg2,python3-psycopg2,python-ipaddress,acl
        state: present
        update_cache: yes
    - name: Config. Edit pg_hba configuration file. Add cpca1server access. Открываем доступ с первой ноды сервера CA к базе cpca для пользователя cpca.
      postgresql_pg_hba:
        dest: /etc/postgresql/11/main/pg_hba.conf
        contype: host
        users: cpca
        source: 192.168.1.3
        databases: all
        method: md5
        create: true
    - name: Config. Edit pg_hba configuration file. Add cpca2server access. Открываем доступ со второй ноды сервера CA к базе cpca для пользователя cpca.
      postgresql_pg_hba:
        dest: /etc/postgresql/11/main/pg_hba.conf
        contype: host
        users: cpca
        source: 192.168.1.4
        databases: all
        method: md5
        create: true
    - name: Config. Edit pg_hba configuration file. Add cpca1server access. Открываем доступ с первой ноды сервера CA к базе cpra для пользователя cpra.
      postgresql_pg_hba:
        dest: /etc/postgresql/11/main/pg_hba.conf
        contype: host
        users: cpra
        source: 192.168.1.3
        databases: all
        method: md5
        create: true
    - name: Config. Edit pg_hba configuration file. Add cpca2server access. Открываем доступ со второй ноды сервера CA к базе cpra для пользователя cpra.
      postgresql_pg_hba:
        dest: /etc/postgresql/11/main/pg_hba.conf
        contype: host
        users: cpra
        source: 192.168.1.4
        databases: all
        method: md5
        create: true
    - name: Config. Bash. Edit postgresql.conf for enable all interfaces. Разрешаем входящие подключения к порту 5432 на всех интерфейсах.
      ansible.builtin.shell: |
        echo "listen_addresses = '*'" >> postgresql.conf
        systemctl restart postgresql
      args:
        executable: /bin/bash
        chdir: /etc/postgresql/11/main/
    - name: Config. Give the postgres user rights to read the mandatory access control database
      ansible.builtin.shell: |
        usermod -a -G shadow postgres
        setfacl -d -m u:postgres:r /etc/parsec/macdb
        setfacl -R -m u:postgres:r /etc/parsec/macdb
        setfacl -m u:postgres:rx /etc/parsec/macdb
        setfacl -d -m u:postgres:r /etc/parsec/capdb
        setfacl -R -m u:postgres:r /etc/parsec/capdb
        setfacl -m u:postgres:rx /etc/parsec/capdb
        pdpl-user -l 0:0 cpca
        pdpl-user -l 0:0 cpra
        systemctl restart postgresql
      args:
        executable: /bin/bash
- name: PostgreSQL | Primary Server. Create database "cpca" and role "cpca" on the Primary.
  hosts: psql1server
  become: true
  become_user: postgres
  vars:
    allow_world_readable_tmpfiles: true
  tasks:
    - name: Create "cpca" user, and grant access to bases create.
      community.postgresql.postgresql_user:
        name: cpca
        password: P@ssw0rd
        expires: "infinity"
        role_attr_flags: "CREATEDB"
    - name: Create a new database with name "cpca".
      community.postgresql.postgresql_db:
        name: cpca
        owner: cpca
        comment: "CryptoPro CA database"
    - name: Connect to "cpca" database, grant privileges on "cpca" database objects (database) for "cpca" role.
      community.postgresql.postgresql_privs:
        database: cpca
        state: present
        privs: ALL
        type: database
        roles: cpca
        grant_option: true
    - name: Connect to "cpca" database, grant privileges on "cpca" database objects (schema) for "cpca" role.
      community.postgresql.postgresql_privs:
        database: cpca
        state: present
        privs: CREATE
        type: schema
        objs: public
        roles: cpca
- name: PostgreSQL | Primary Server. Create database "cpra" and role "cpra" on the Primary.
  hosts: psql1server
  become: true
  become_user: postgres
  vars:
    allow_world_readable_tmpfiles: true
  tasks:
    - name: Create "cpra" user, and grant access to bases create.
      community.postgresql.postgresql_user:
        name: cpra
        password: P@ssw0rd
        expires: "infinity"
        role_attr_flags: "CREATEDB"
    - name: Create a new database with name "cpra".
      community.postgresql.postgresql_db:
        name: cpra
        owner: cpra
        comment: "CryptoPro RA database"
    - name: Connect to "cpra" database, grant privileges on "cpra" database objects (database) for "cpra" role.
      community.postgresql.postgresql_privs:
        database: cpra
        state: present
        privs: ALL
        type: database
        roles: cpra
        grant_option: true
    - name: Connect to cpra database, grant privileges on "cpra" database objects (schema) for "cpra" role.
      community.postgresql.postgresql_privs:
        database: cpra
        state: present
        privs: CREATE
        type: schema
        objs: public
        roles: cpra
```
</details>


#### Распаковка дистрибутивов __КриптоПРО__ в рабочий каталог, ввод лицензий и настройка гаммы на сервере __cpca1server__
Все дальнейшие шаги мы также будем выполнять удалённо со станции администратора (в нашем случае это хост виртуализации), теперь уже с помощью плейбуков _Ansible_.

Распакуем архив с дистрибутивом _КриптоПро CSP_, установим все необходимые пакеты и __гамму__, 
которая была предварительно сгенерирована в _КриптоПро CSP_ на рабочей станции _Windows_. _Ansible-playbook_ в данном случае выглядит так:

<details>
<summary>Ansible code</summary>

```
---
- name: CryptoPro CSP | 1. Extract archives. Install distributives CryptoPro CSP, Stunnel, Nginx, PKI Cades. Setup Licenses for CSP, OCSP, TSP. Install Gamma.
  hosts: cpservers
  tasks:

    - name: CryptoPro CSP. Extract archive
      ansible.builtin.shell: |
        test -d csp50r2 || mkdir $_
        tar -C csp50r2 -xvzf linux-amd64_deb.tgz --strip-components 1
      args:
        executable: /bin/bash
        chdir: /home/vagrant/

- name: CryptoPro CSP | Install distributives CryptoPro CSP, Stunnel, Nginx, PKI Cades. Setup Licenses for CSP, OCSP, TSP. Install Gamma.
  hosts: cpservers
  become: true
  tasks:
    - name: CryptoPro CSP. Install Software CryptoPro CSP, Stunnel, Nginx, PKI Cades. Setup Licenses for CSP, OCSP, TSP. Install Gamma.
      ansible.builtin.shell: |
        csp50r2/install.sh
        dpkg -i csp50r2/lsb-cprocsp-kc2-64_5.0.13000-7_amd64.deb #КС2
        dpkg -i csp50r2/cprocsp-stunnel-64_5.0.13000-7_amd64.deb #stunnel
        dpkg -i csp50r2/lsb-cprocsp-devel_5.0.13000-7_all.deb #devel
        dpkg -i csp50r2/cprocsp-nginx-64_5.0.13000-7_amd64.deb #nginx
        dpkg -i csp50r2/cprocsp-pki-cades-64_2.0.15000-1_amd64.deb #cades
        /opt/cprocsp/sbin/amd64/cpconfig -license -set 'XXXXX-XXXXX-XXXXX-XXXXX-XXXXX'
        /opt/cprocsp/bin/amd64/ocsputil li -s 'XXXXX-XXXXX-XXXXX-XXXXX-XXXXX'
        /opt/cprocsp/bin/amd64/tsputil li -s 'XXXXX-XXXXX-XXXXX-XXXXX-XXXXX'
        cp Crypto\ Pro/db1/kis_1 /var/opt/cprocsp/dsrf/db1/
        cp Crypto\ Pro/db2/kis_1 /var/opt/cprocsp/dsrf/db2/
        chmod -R 777 "/var/opt/cprocsp/dsrf/"
        /opt/cprocsp/sbin/amd64/cpconfig -hardware rndm -add cpsd -name 'cpsd rng' -level 3
        /opt/cprocsp/sbin/amd64/cpconfig -hardware rndm -configure cpsd -add string /db1/kis_1 /var/opt/cprocsp/dsrf/db1/kis_1
        /opt/cprocsp/sbin/amd64/cpconfig -hardware rndm -configure cpsd -add string /db2/kis_1 /var/opt/cprocsp/dsrf/db2/kis_1
        systemctl restart cprocsp.service
      args:
        executable: /bin/bash
        chdir: /home/vagrant/
```
</details>

В задаче __"CryptoPro CSP. Extract archive"__ мы распаковываем архив с установочными файлами в предварительно созданный каталог в домашней директории пользователя _Vagrant_.
Затем с помощью задачи __"CryptoPro CSP. Install Software CryptoPro CSP, Stunnel, Nginx, PKI Cades. Setup Licenses for CSP, OCSP, TSP. Install Gamma"__ устанавливаем пакеты и копируем 
_гамму_ в рабочий каталог.
> [!NOTE]
> Лицензии, которые необходимо ввести на данном шаге, либо приобретаются, либо запрашиваются у вендора в качестве ознакомительных.

#### Установка программного обеспечения, создание группы безопасности и пользователей на сервере __cpca1server__

Код плейбука под спойлером:

<details>
<summary>Ansible code</summary>

```
---
- name: CryptoPro CA | 2. Install packages acl,postgresql-client. Create group and users. Create .pgpass file
  hosts: cpcaserver
  become: true
  tasks:

    - name: CryptoPro CA. APT. Update the repository cache and install packages "acl" and "postgresql-client" to latest version
      ansible.builtin.apt:
        name: acl,postgresql-client
        state: present
        update_cache: false

    - name: CryptoPro CA. Download and install old libssl1.0 package (Only for Debian Linux)
      ansible.builtin.shell: |
        # wget http://snapshot.debian.org/archive/debian/20190501T215844Z/pool/main/g/glibc/multiarch-support_2.28-10_amd64.deb
        # wget https://snapshot.debian.org/archive/debian-archive/20190328T105444Z/debian/pool/main/o/openssl/libssl1.0.0_1.0.2l-1~bpo8%2B1_amd64.deb
        # dpkg -i multiarch-support_2.28-10_amd64.deb
        # dpkg -i libssl1.0.0_1.0.2l-1~bpo8+1_amd64.deb
      args:
        executable: /bin/bash
        chdir: /home/vagrant/

    - name: CryptoPro CA. Ensure group "crl-writers" exists
      ansible.builtin.group:
        name: crl-writers
        state: present

    - name: CryptoPro CA. Add the user 'cpca' with a bash shell, appending the group 'crl-writers' and to the user's groups
      ansible.builtin.user:
        name: cpca
        password: '!'
        comment: CryptoPro CA
        shell: /bin/bash
        create_home: yes
        groups: crl-writers
        append: yes

    - name: CryptoPro CA. Add the user 'cpra' with a bash shell
      ansible.builtin.user:
        name: cpra
        password: '!'
        comment: CryptoPro RA
        shell: /bin/bash
        create_home: yes

    - name: CryptoPro CA. Create .pgpass file for new users
      ansible.builtin.shell: |
        cat .pgpass > /home/cpca/.pgpass
        cat .pgpass > /home/cpra/.pgpass
        chown cpca: /home/cpca/.pgpass
        chown cpra: /home/cpra/.pgpass
        chmod 600 /home/cpca/.pgpass
        chmod 600 /home/cpra/.pgpass
      args:
        executable: /bin/bash
        chdir: /home/vagrant/
```
</details>

В данном примере мы с помощью утилиты _Apt_ установили из репозитория _Astra Linux_ пакеты _acl_ и _postgresql-client_, создали группу пользователей _crl-writers_ 
и две учётные записи - _cpca_ и _cpra_.  С правами первой будут работать сервисы _NATS_ и _CryptoPro.Ca.Service_. Учётная запись _cpra_ нужна для 
работы служб _CryptoPro.Ra.Service_ и _CryptoPro.Ra.Web_.

> [!TIP]
> В качестве репозитория программного обеспечения будем использовать установочный носитель __Astra Linux 1.7.5__, предварительно смонтированный для чтения внутри виртуальной машины. 
> Диск монтруется с помощью команды `mount -o loop /home/vagrant/installation-1.7.5.9-16.10.23_16.58.iso /media/localiso/` вызываемой в блоке _SHELL_ соответствующего _Vagrantfile_.
> Также потребуется добавить в файл _/etc/apt/sources.list_ строку `deb file:///media/localiso/ 1.7_x86-64 contrib main non-free`, а имеющиеся описания сетевых репозиториев
> вида `deb https://download.astralinux.ru/astra/stable/1.7_x86-64/repository-main/ 1.7_x86-64 main contrib non-free` закрыть символом комментария.

В заключение сценария в домашнем каталоге каждого созданного пользователя создаются файлы _.pgpass_ для подключения к базам данных сервера _Postgresql_.

> [!NOTE] 
> Задача _CryptoPro CA. Download and install old libssl1.0 package (Only for Debian Linux)_ предназначена для скачивания и установки устаревшего пакета _libssl1.0_
> в случае разворачивания комплекса _КриптоПро УЦ 2.0_ на современных дистрибутивах _Debian_, новее выпуска _Jessie_.

#### Установка и настройка приложений _CryptoPro.Ca.Service_, _CryptoPro.Ra.Service_, _CryptoPro.Ra.Web_, _NATS_

Следующий шаг - создание и настройка основных пакетов _Удостоверяющего Центра_. Для этого понадобится следующий _playbook_:

<details>
<summary>Ansible code</summary>

```
---
- name: CryptoPro CA | 3. Install package CPCA. Set ACL privileges for pkica and CryptoPro.Ca.Service. Edit main config CPCA and CryptoPro.Ca.Service
  hosts: cpcaserver
  become: true
  tasks:

    - name: CryptoPro CA. APT. Install package "unzip" to latest version
      ansible.builtin.apt:
        name: unzip
        state: present
        update_cache: false

    - name: CryptoPro CA. Extract archive.
      ansible.builtin.shell: |
        test -d /opt/cpca || mkdir $_
        unzip ca-linux-x64-1.63.0.32.zip -d /opt/cpca
      args:
        executable: /bin/bash
        chdir: /home/vagrant/

    - name: CryptoPro CA. Set ACL privileges for CryptoPro.Ca.Service, CryptoPro.Ra.Service, CryptoPro.Ra.Web
      ansible.builtin.shell: |
        setfacl -m "u:cpca:r-x" CryptoPro.Ca.Service/CryptoPro.Ca.Service
        setfacl -m "u:cpra:r-x" CryptoPro.Ra.Service/CryptoPro.Ra.Service
        setfacl -m "u:cpra:r-x" CryptoPro.Ra.Web/CryptoPro.Ra.Web
        setfacl -m "u:cpca:r-x" pkica/pkica
        setfacl -m "u:cpra:r-x" pkica/pkica
        setfacl -m "u:cpca:rwx" /home/cpra
      args:
        executable: /bin/bash
        chdir: /opt/cpca/

    - name: CryptoPro CA. Set ACL privileges for certmgr, cryptcp
      ansible.builtin.shell: |
        setfacl -m "u:cpca:r-x" certmgr
        setfacl -m "u:cpra:r-x" certmgr
        setfacl -m "u:cpca:r-x" cryptcp
        setfacl -m "u:cpra:r-x" cryptcp
      args:
        executable: /bin/bash
        chdir: /opt/cprocsp/bin/amd64/

    - name: CryptoPro CA. Set ACL privileges for nats-streaming-server (nats-streaming-server - Служба очередей NATS Streaming с поддержкой ГОСТ TLS)
      ansible.builtin.shell: |
        setfacl -m "u:cpca:r-x" nats-streaming-server
      args:
        executable: /bin/bash
        chdir: /opt/cpca/nats-streaming/

    - name: CryptoPro CA. Edit main config for pkica (pkica - Программа настройки УЦ)
      ansible.builtin.shell: |
        sed -i '/CaDb/{n;n;s/Server=localhost;Database=Ca;Username=postgres;Pooling=True/Server=psql1server;Database=cpca;Username=cpca;Pooling=True/}' appsettings.json
        sed -i '/RaDb/{n;n;s/Server=localhost;Database=Ra;Username=postgres;Pooling=True/Server=psql1server;Database=cpra;Username=cpra;Pooling=True/}' appsettings.json
        sed -i 's/"Secure": true/"Secure": false/' appsettings.json
        sed -i "/Nats/{n;s/localhost/$HOSTNAME/}" appsettings.json
        sed -i "/Stan/{n;s/localhost/$HOSTNAME/}" appsettings.json
      args:
        executable: /bin/bash
        chdir: /opt/cpca/pkica/

    - name: CryptoPro CA. Edit main config (CryptoPro.Ca.Service - Сервис ЦС)
      ansible.builtin.shell: |
        sed -i 's/Server=localhost;Database=Ca;Username=postgres;Pooling=True/Server=psql1server;Database=cpca;Username=cpca;Pooling=True/' appsettings.json
        sed -i '/nats/{n;n;s/"Secure": true/"Secure": false/}' appsettings.json
        sed -i '/ClientID/{n;n;s/"Secure": true/"Secure": false/}' appsettings.json
        sed -i "/Nats/{n;s/localhost/$HOSTNAME/}" appsettings.json
        sed -i "/Stan/{n;s/localhost/$HOSTNAME/}" appsettings.json
        sed -i 's/"SerialNumber": ""/"SerialNumber": "XXXXX-XXXXX-XXXXX-XXXXX-XXXXX"/' appsettings.json
        sed -i 's/"Company": ""/"Company": "ОАО \\"ЦЕНТР РАЗУМНЫХ РЕШЕНИЙ\\" ЛИМИТЕД"/' appsettings.json
      args:
        executable: /bin/bash
        chdir: /opt/cpca/CryptoPro.Ca.Service/

    - name: CryptoPro RA. Edit main config (CryptoPro.Ra.Service - Сервис ЦР)
      ansible.builtin.shell: |
        export PATH=$PATH:/opt/cprocsp/bin/amd64
        ttoldra=$(grep Thumbprint appsettings.json | tail -n 1 | awk '{print $2}' | awk -F "," '{print $1}')
        ttnewra=$(certmgr -list -file /home/cpra/raclnt.cer | grep SHA1 | awk '{print $4}')
        sed -i 's/Server=localhost;Database=Ra;Username=postgres;Pooling=True/Server=psql1server;Database=cpra;Username=cpra;Pooling=True/' appsettings.json
        sed -i "/Nats/{n;s/localhost/$HOSTNAME/}" appsettings.json
        sed -i "/Stan/{n;s/localhost/$HOSTNAME/}" appsettings.json
        sed -i 's/"SerialNumber": ""/"SerialNumber": "XXXXX-XXXXX-XXXXX-XXXXX-XXXXX"/' appsettings.json
        sed -i 's/"Company": ""/"Company": "ОАО \\"ЦЕНТР РАЗУМНЫХ РЕШЕНИЙ\\" ЛИМИТЕД"/' appsettings.json
      args:
        executable: /bin/bash
        chdir: /opt/cpca/CryptoPro.Ra.Service/

    - name: CryptoPro CA. Edit nats-streaming-server daemon config (nats-streaming-server - Служба очередей NATS Streaming с поддержкой ГОСТ TLS)
      ansible.builtin.shell: |
        sed -i "s/ca.example/$HOSTNAME/" nats.no-tls.conf
        sed -i "s/ca.example/$HOSTNAME/" nats.conf
      args:
        executable: /bin/bash
        chdir: /opt/cpca/nats-streaming/
```
</details>

Здесь мы с помощью задач: 
  - __"CryptoPro CA. APT. Install package "unzip" to latest version"__ - установили пакет _unzip_ ;
  - __"CryptoPro CA. Extract archive"__ - распаковали архив с приложениями;
  - __"CryptoPro CA. Set ACL privileges for CryptoPro.Ca.Service, CryptoPro.Ra.Service, CryptoPro.Ra.Web"__ - разрешили запуск приложений от имени соответствующих учётных записей;
  - __"CryptoPro CA. Set ACL privileges for certmgr, cryptcp"__ - разрешили запуск утилит от имени соответствующих учётных записей (необязательное действие - соответствующие файлы по 
умолчанию имеют бит исполнения для всех категорий пользователей);
  - __"CryptoPro CA. Set ACL privileges for nats-streaming-server (nats-streaming-server - Служба очередей NATS Streaming с поддержкой ГОСТ TLS)"__ - разрешили 
запуск сервиса для пользователя _cpca_;
  - __"CryptoPro CA. Edit main config for pkica (pkica - Программа настройки УЦ)"__ - настроили подключение к базам данных _Центра Сертификации_ и _Центра Регистрации_, отключили использование
протокола _TLS_ и задали имя хоста служб _Nats_, _Stan_ для _pkica_;
  - __"CryptoPro CA. Edit main config (CryptoPro.Ca.Service - Сервис ЦС)"__ - настроили подключение к базе данных _Центра Сертификации_, отключили использование протокола 
_TLS_, задали имя хоста служб _Nats_, _Stan_ для _CryptoPro.Ca.Service_ и активировали лицензию.
  - __"CryptoPro RA. Edit main config (CryptoPro.Ra.Service - Сервис ЦР)"__ _ проделали все те же шаги, что и в предыдущем пункте, но для службы _CryptoPro.Ra.Service_.

#### Создание рабочих каталогов, регистрация новых сервисов

На данном этапе мы создадим каталоги:
  - для хранения очередей (сообщений) - _/var/lib/nats-streaming/pkica-store_;
  - для хранения сертификатов __nats-streaming-server__ - _/opt/cpca/nats-streaming/ssl/_;
  - для публикации __САС__ (список аннулированных сертификатов) - _/var/lib/cpca/cdp_;
  - для хранения журналов __Центра Сертификации__ - _/var/log/cpca_.

Предоставим группе безопасности _crl-writers_ права для установки CRL в хранилище _LocalMachine\CA_ (файл _/var/opt/cprocsp/users/stores/ca.sto_).

> Для подключения к NATS/NATS Streaming с использованием TLS необходимо, чтобы в локальном хранилище компьютера «Промежуточные Центры сертификации» (Local Machine\CA)
> были установлены действующие CRL. Сервис ЦС (CryptoPro.Ca.Service) и другие сервисы, взаимодействующие с ним через NATS Streaming, поддерживают автоматическую установку 
> выпущенных CRL в это хранилище.[^1]

Зарегистрируем, разрешим загрузку и запустим сервисы _SystemD_:
  - ![nats-streaming-server.service](files/cprokey/nats-streaming-server.service);
  - ![cryptopro.ca.service](files/cprokey/cryptopro.ca.service);
  - ![cryptopro.ra.service](files/cprokey/cryptopro.ra.service).

Примеры файлов служб: nats-streaming-server.service:

```
[Unit]
Description=nats-streaming-server
[Service]
WorkingDirectory=/opt/cpca/nats-streaming
ExecStart=/opt/cpca/nats-streaming/nats-streaming-server -sc /opt/cpca/nats-streaming/nats.no-tls.conf
Restart=always
# Restart service after 10 seconds if the dotnet service crashes:
RestartSec=10
KillSignal=SIGINT
SyslogIdentifier=nats-streaming-server
User=cpca
[Install]
WantedBy=multi-user.target
```

cryptopro.ca.service:

```
[Unit]
Description=CryptoPro.Ca.Service
[Service]
WorkingDirectory=/opt/cpca/CryptoPro.Ca.Service/
ExecStart=/opt/cpca/CryptoPro.Ca.Service/CryptoPro.Ca.Service
Restart=always
# Restart service after 10 seconds if the dotnet service crashes:
RestartSec=10
KillSignal=SIGINT
SyslogIdentifier=cryptopro-ca-service
User=cpca
[Install]
WantedBy=multi-user.target
```

cryptopro.ra.service:

```
[Unit]
Description=CryptoPro.Ra.Service
[Service]
WorkingDirectory=/opt/cpca/CryptoPro.Ra.Service/
ExecStart=/opt/cpca/CryptoPro.Ra.Service/CryptoPro.Ra.Service
Restart=always
# Restart service after 10 seconds if the dotnet service crashes:
RestartSec=10
KillSignal=SIGINT
SyslogIdentifier=cryptopro-ra-service
User=cpra
[Install]
WantedBy=multi-user.target
```

Код плейбука под спойлером:
<details>
<summary>Ansible code</summary>

```
---
- name: CryptoPro CA | 4. Create a messages store dir for nats-streaming-server. Create directories. Register nats-streaming-server daemon
  hosts: cpcaserver
  become: true
  tasks:

    - name: CryptoPro CA. Create a messages store dir for nats-streaming-server
      ansible.builtin.shell: |
        mkdir -p /var/lib/nats-streaming/pkica-store
        chown -R cpca: /var/lib/nats-streaming/pkica-store
      args:
        executable: /bin/bash

    - name: CryptoPro CA. Create a certificates store dir for nats-streaming-server
      ansible.builtin.shell: |
        mkdir -p /opt/cpca/nats-streaming/ssl/
        chown -R cpca: /opt/cpca/nats-streaming/ssl/
      args:
        executable: /bin/bash

    - name: CryptoPro CA. Create a crl store dir for ca-server
      ansible.builtin.shell: |
        mkdir -p /var/lib/cpca/cdp
        chown -R cpca: /var/lib/cpca/cdp
      args:
        executable: /bin/bash

    - name: CryptoPro CA. Create a log dir for cryptopro.ca.service
      ansible.builtin.shell: |
        mkdir -p /var/log/cpca
        chown -R cpca: /var/log/cpca
      args:
        executable: /bin/bash

    - name: CryptoPro CA. Change owner for storage LocalMachine\CA
      ansible.builtin.shell: |
        chown :crl-writers ca.sto
        chmod g+w ca.sto
      args:
        executable: /bin/bash
        chdir: /var/opt/cprocsp/users/stores/

    - name: CryptoPro CA. Configure and Register nats-streaming-server cryptopro.ca.service, cryptopro.ra.service, start services
      ansible.builtin.shell: |
        cp nats-streaming-server.service /etc/systemd/system/
        cp cryptopro.ca.service /etc/systemd/system/
        cp cryptopro.ra.service /etc/systemd/system/
        chown root: /etc/systemd/system/nats-streaming-server.service /etc/systemd/system/cryptopro.ca.service /etc/systemd/system/cryptopro.ra.service
        systemctl daemon-reload
        systemctl enable nats-streaming-server.service
        systemctl enable cryptopro.ca.service
        systemctl enable cryptopro.ra.service
        systemctl start nats-streaming-server.service
        systemctl start cryptopro.ca.service
        systemctl start cryptopro.ra.service
      args:
        executable: /bin/bash
        chdir: /home/vagrant/
```
</details>

[^1]: 2.6 Установка CRL в хранилище. ЖТЯИ.00078-03 90 02. ПАК КриптоПро УЦ 2.0. Руководство по установке.
