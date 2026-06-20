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
allow_world_readable_tmpfiles: true
liccsp: XXXXX-XXXXX-XXXXX-XXXXX-XXXXX
licocsputils: XXXXX-XXXXX-XXXXX-XXXXX-XXXXX
lictsputils: XXXXX-XXXXX-XXXXX-XXXXX-XXXXX
csp_distro: linux-amd64_deb.tgz
src_dirinst: /home/max/vagrant/ansible.ca/files
dst_dirinst: /home/vagrant/ansible.ca/files
dst_user: vagrant
localdir: /home/max/vagrant/ansible.ca
dircsp: csp50r2
cspbase: /opt/cprocsp/
cspbindir: /opt/cprocsp/bin/amd64/
cspsbindir: /opt/cprocsp/sbin/amd64/
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
  :white_check_mark: extract
  :white_check_mark: install
  :white_check_mark: license
  :white_check_mark: gamma
  :white_check_mark: restart

