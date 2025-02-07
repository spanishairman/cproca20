#### Установка и первоначальная настройка КриптоПро ЦА 2.0 для Linux (сборка 1.63.0.32) в среде виртуализации 

Пример установки и первоначальной настройки удостоверяющего центра на базе программного обеспечения от _КриптоПро_. В данном примере попытаемся развернуть УЦ для тестирования с использованием среды виртуализации.
> [!IMPORTANT]
> Отметим сразу, что рассматриваемая здесь установка **КриптоПро ЦА 2.0** **не подразумевает** использование его в промышленных целях, так как мы не будем следовать требованиям, 
> содержащимся в эксплуатационной документации и предъявляемым при работе со _СКЗИ_ со стороны действующего законодательства.

Стенд, на котором будет развернут комплекс, представляет собой хост-машину, где в качестве гипервизора установлено ПО виртуализации _KVM_ и среда разработки - _Vagrant_.
Все узлы _Удостоверяющего Центра_ - это виртуальные машины, которые разворачиваются с помощью средств автоматизации и оркестрации _Vagrant_ и _Ansible_.

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
      chmod 600 ~/.pgpass
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

Здесь приведено описание двух виртуальных машин: _cpca1server_ и _psql1server_ - сервер _Центра сертивикации/Центра регистрации_ и сервер баз данных, соответственно. В качестве _СУБД_ используется 
_PostgreSQL_, установленная из репозиториев операционной системы. В соответствии с прилагаемым к _КриптоПро УЦ_ формуляром, будем разворачивать наш УЦ в ОС специального назначения - _Astra linux_.

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
В данном примере мы загрузили на виртуальную машину __cpca1server__, выступающую в роли УЦ, дистрибутивы: 
  - _"КриптоПро УЦ 2.0 для Linux (сборка 1.63.0.32)"_ - ca-linux-x64-1.63.0.32.zip; 
  - _КриптоПро CSP_ версии 5.0.13000-7 - linux-amd64_deb.tgz;
  - Файл паролей для подключения к базам данных _PostgreSQL_ - pgpass;
  - Файлы конфигурации служб для демона _SystemD_ - cryptopro.ca.service, cryptopro.ra.service, nats-streaming-server.service;
  - Каталог с ранее сформированной _гаммой_ - Crypto Pro.
