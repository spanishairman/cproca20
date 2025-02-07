#### Установка и первоначальная настройка КриптоПро УЦ 2.0 для Linux (сборка 1.63.0.32) в среде виртуализации 

Пример установки и первоначальной настройки удостоверяющего центра на базе программного обеспечения от _КриптоПро_. В данном примере попытаемся, используя средства виртуализации, 
развернуть УЦ от [КриптоПро](https://cryptopro.ru/products/ca/2.0) для ознакомления с функционалом и тестирования.
> [!IMPORTANT]
> Отметим сразу, что рассматриваемая здесь установка _КриптоПро УЦ 2.0_ **не подразумевает** использование его в промышленных целях, так как мы не будем следовать множеству требованиий, 
> содержащихся в эксплуатационной документации и подлежащих исполнению при работе со _СКЗИ_ со стороны действующего законодательства.

Стенд, на котором будет развернут комплекс, представляет собой хост-машину, где в качестве гипервизора установлено ПО виртуализации _KVM_ и среда разработки - _Vagrant_.
Все узлы _Удостоверяющего Центра_ - это _QEMU_ образы виртуальных машин, которые разворачиваются с помощью средств автоматизации и оркестрации _Vagrant_ и _Ansible_.

##### Установка и первоначальная настройка виртуальных машин с помощью _Vagrant_
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

Здесь приведено описание двух виртуальных машин: _cpca1server_ и _psql1server_ - сервер _Центра сертификации/Центра регистрации_ и сервер баз данных, соответственно. 
В качестве _СУБД_ используется _PostgreSQL_, установленная из репозиториев операционной системы. 
В соответствии с прилагаемым к _КриптоПро УЦ_ формуляром, будем разворачивать наш УЦ в ОС специального назначения - _Astra linux 1.7.5_.

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

Далее, в блоке настроек _SHELL_ мы редактируем файл _/etc/hosts_ и задаём базовые правила и политики для _iptables_.

Все создаваемые виртуальные машины имеют два сетевых интерфейса: 
  - _vagrant-libvirt-inet1_ - изолированная сеть, без выхода в интернет и доступа к каким-либо хостам, включая хост виртуализации;
  - _vagrant-libvirt-mgmt_ - сеть управления, с её помощью выполняется управление виртуальными машинами, а также, эта сеть позволяет виртуальным машинам выходить в интернет. 

##### Распаковка дистрибутивов __КриптоПРО__ в рабочий каталог, ввод лицензий и настройка гаммы на сервере __cpca1server__
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

Здесь, в задаче _CryptoPro CSP. Extract archive_ мы распаковываем архив с установочными файлами в предварительно созданный каталог в домашней директории пользователя _Vagrant_.
Затем, с помощью задачи _CryptoPro CSP. Install Software CryptoPro CSP, Stunnel, Nginx, PKI Cades. Setup Licenses for CSP, OCSP, TSP. Install Gamma_, устанавливаем пакеты и копируем 
_гамму_ в рабочий каталог.
> [!NOTE]
> Лицензии, которые необходимо ввести на данном шаге, либо приобретаются у вендора, либо запрашиваются в качестве ознакомительных.

##### Установка программного обеспечения, создание группы и пользователей на сервере __cpca1server__

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

В данном примере мы с помощью утилиты _Apt_ установили из репозитория _Astra Linux_ пакеты _acl_ и _postgresql-client_, создали группу пользователей _crl-writers_ и две учётные записи - _cpca_ и _cpra_. 
С правами первой будут работать сервисы _NATS_ и _CryptoPro.Ca.Service_. Учётная запись _cpra_ нужна для работы службы _CryptoPro.Ra.Service_ и _CryptoPro.Ra.Web_.

> [!TIP]
> В качестве репозитория программного обеспечения будем использовать установочный носитель __Astra Linux 1.7.5__, предварительно смонтированный для чтения внутри виртуальной машины. 
> Диск монтруется с помощью команды `mount -o loop /home/vagrant/installation-1.7.5.9-16.10.23_16.58.iso /media/localiso/` вызываемой в блоке _SHELL_ соответствующего _Vagrantfile_.
> Также потребуется добавить в файл _/etc/apt/sources.list_ строку `deb file:///media/localiso/ 1.7_x86-64 contrib main non-free`, а имеющиеся описания сетевых репозиториев
> вида `deb https://download.astralinux.ru/astra/stable/1.7_x86-64/repository-main/ 1.7_x86-64 main contrib non-free` закрыть символом комментария.

В заключение сценария домашнем каталоге каждого созданного пользователя создаются файлы _.pgpass_ для подключения к базам данных сервера _Postgresql_.

> [!NOTE] Задача _CryptoPro CA. Download and install old libssl1.0 package (Only for Debian Linux)_ предназначена для скачивания и установки устаревшего пакета _libssl1.0_
> в случае разворачивания комплекса _КриптоПро УЦ 2.0_ на современных дистрибутивах _Debian_, новее выпуска _Jessie_.

##### Установка и настройка сервисов _CryptoPro.Ca.Service_, _CryptoPro.Ra.Service_, _CryptoPro.Ra.Web_

Следующий шаг - создание и настройка основных служб _Центра Сертификации_. Для этого понадобится следующий _playbook_:

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
  - _"CryptoPro CA. APT. Install package "unzip" to latest version"_ - установили пакет _unzip_ ;
  - _"CryptoPro CA. Extract archive"_ - распаковали архив с приложениями;
  - _"CryptoPro CA. Set ACL privileges for CryptoPro.Ca.Service, CryptoPro.Ra.Service, CryptoPro.Ra.Web"_ - разрешили запуск сервисов от имени соответствующих учётных записей;
  - _CryptoPro CA. Set ACL privileges for certmgr, cryptcp_ - разрешили запуск сервисов от имени соответствующих учётных записей (необязательно, соответствующие файлы по 
умолчанию имеют бит исполнения для всех категорий пользователей);
  - _CryptoPro CA. Set ACL privileges for nats-streaming-server (nats-streaming-server - Служба очередей NATS Streaming с поддержкой ГОСТ TLS)_ - разрешили 
запуск сервиса для пользователя _cpca_;
  - _CryptoPro CA. Edit main config for pkica (pkica - Программа настройки УЦ)_ - настроили подключение к базам данных _Центра Сертификации_ и _Центра Регистрации_, отключили использование
протокола _TLS_ и задали имя хоста служб _Nats_, _Stan_ для pkica - программы настройки УЦ.
