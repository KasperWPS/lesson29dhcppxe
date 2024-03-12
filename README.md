# Домашнее задание № 19 по теме: "DHCP, PXE". К курсу Administrator Linux. Professional

## Задание

- Развернуть сервер на CentOS 8 со следующими сервисами:
  - TFTPD
  - DHCP
  - HTTPD
- Реализовать автоматическую установку (kickstart)
- ***** Самостоятельно развернуть Cobbler (https://cobbler.readthedocs.io/en/latest/quickstart-guide.html)

## Выполнение

- ОС: CentOS/8
- Vagrant 2.4.0
- Ansible 2.16.4
- Cobbler 3.2.2

Конфигурация (**config.json**):

- pxeserver - стенд для задания без *
- pxeclient - стенд для проверки всех заданий, загружается по сети, имеет GUI
- cobbler - стенд с настроенным cobbler

```json
[
  {
    "name": "pxeserver",
    "cpus": 1,
    "gui": false,
    "box": "centos/8",
    "private_network":
    [
      { "ip": "10.0.0.20",      "adapter": 2, "netmask": "255.255.255.0", "virtualbox__intnet": "pxenet" },
      { "ip": "192.168.56.200", "adapter": 3, "netmask": "255.255.255.0" }
    ],
    "memory": "1024",
    "no_share": true,
    "disks": {
      "sata1": {
        "dfile": "./disks/sata1.vdi",
        "size": "20000",
        "ctlname": "SATA Controller"
      }
    },
    "controllers":
    [
      { "name": "SATA Controller", "ctl": "sata" }
    ]
  },
  {
    "name": "pxeclient",
    "cpus": 1,
    "gui": true,
    "box": "centos/8",
    "private_network":
    [
      { "ip": "10.0.0.21",      "adapter": 2, "netmask": "255.255.255.252", "virtualbox__intnet": "pxenet" },
      { "ip": "192.168.56.210", "adapter": 3, "netmask": "255.255.255.0" }
    ],
    "memory": 8192,
    "no_share": true,
    "lanboot": true
  },
  {
    "name": "cobbler",
    "cpus": 1,
    "gui": false,
    "box": "centos/8",
    "private_network":
    [
      { "ip": "10.0.0.22",      "adapter": 2, "netmask": "255.255.255.0", "virtualbox__intnet": "pxenet" },
      { "ip": "192.168.56.220", "adapter": 3, "netmask": "255.255.255.0" }
    ],
    "memory": "1024",
    "no_share": true,
    "disks": {
      "sata1": {
        "dfile": "./disks/sata-cobbler.vdi",
        "size": "20000",
        "ctlname": "SATA Controller"
      }
    },
    "controllers":
    [
      { "name": "SATA Controller", "ctl": "sata" }
    ]
  }
]

```
*Примечание: контроллеры можно исключить если они присутствуют по-умолчанию (SATA)*


Vagrantfile:

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby : vsa
Vagrant.require_version ">= 2.2.17"

class Hash
  def rekey
  t = self.dup
  self.clear
  t.each_pair{|k, v| self[k.to_sym] = v}
    self
  end
end

require 'json'

f = JSON.parse(File.read(File.join(File.dirname(__FILE__), 'config.json')))
# Локальная переменная PATH_SRC для монтирования
$PathSrc = ENV['PATH_SRC'] || "."

Vagrant.configure(2) do |config|
  if Vagrant.has_plugin?("vagrant-vbguest")
    config.vbguest.auto_update = false
  end

  # включить переадресацию агента ssh
  config.ssh.forward_agent = true
  # использовать стандартный для vagrant ключ ssh
  config.ssh.insert_key = false

  f.each do |g|

    config.vm.define g['name'] do |s|
      s.vm.box = g['box']
      s.vm.hostname = g['name']

      if g['private_network']
        g['private_network'].each do |ni|
          s.vm.network "private_network", **ni.rekey
        end
      end

      s.vm.synced_folder $PathSrc, "/vagrant", disabled: g['no_share']

      s.vm.provider :virtualbox do |virtualbox|
        virtualbox.customize [
          "modifyvm",             :id,
          "--audio",              "none",
          "--cpus",               g['cpus'],
          "--memory",             g['memory'],
          "--graphicscontroller", "VMSVGA",
          "--vram",               "64"
        ]

        if g['lanboot']
          virtualbox.customize [
            'modifyvm',:id,
            '--boot1', 'net',
            '--boot2', 'none',
            '--boot3', 'none',
            '--boot4', 'none'
          ]
        end

        if g['controllers']
          g['controllers'].each do |ctlconf|
            virtualbox.customize [
              "storagectl", :id,
              "--name",     ctlconf['name'],
              "--add",      ctlconf['ctl']
            ]
          end
        end

        port = 0

        if g['disks']
          g['disks'].each do |dname, dconf|
            unless File.exist? (dconf['dfile'])
              virtualbox.customize [
                'createhd',
                '--filename', dconf['dfile'],
                '--variant',  'Fixed',
                '--size',     dconf['size']
              ]
            end
          end
          g['disks'].each do |dname, dconf|
            virtualbox.customize [
              'storageattach', :id,
              '--storagectl',  dconf['ctlname'],
              '--port',        port += 1,
              '--device',      0,
              '--type',        'hdd',
              '--medium',      dconf['dfile']
            ]
          end
        end
        virtualbox.gui = g['gui']
        virtualbox.name = g['name']
      end
      if g['name'].eql? "pxeserver"
        s.vm.provision "shell", inline: <<-SHELL
          if [ -b /dev/sdb ] && [ ! -b /dev/sdb1 ]; then
            parted -s /dev/sdb mklabel gpt
            parted /dev/sdb mkpart primary ext4 0% 100%

            if [ $? -eq 0 ]; then
              mkfs.ext4 /dev/sdb1
              partid=`blkid -o export /dev/sdb1 | grep PARTUUID`
              echo "${partid} /mnt ext4 noatime 0 0" >> /etc/fstab
              mount /mnt
              mkdir /mnt/iso
            fi
          fi
        SHELL
      end
      if g['name'].eql? "cobbler"
        s.vm.provision "shell", inline: <<-SHELL
          if [ -b /dev/sdb ] && [ ! -b /dev/sdb1 ]; then
            parted -s /dev/sdb mklabel gpt
            parted /dev/sdb mkpart primary ext4 0% 100%

            if [ $? -eq 0 ]; then
              mkfs.ext4 /dev/sdb1
              partid=`blkid -o export /dev/sdb1 | grep PARTUUID`
              echo "${partid} /mnt ext4 noatime 0 0" >> /etc/fstab
              mount /mnt
              mkdir /mnt/iso
              mkdir /mnt/distro_mirror
            fi
          fi
        SHELL
      end
      s.vm.provision "ansible" do |ansible|
        ansible.playbook = "provisioning/playbook.yml"
        ansible.inventory_path = "provisioning/hosts"
        ansible.host_key_checking = "false"
        ansible.become = "true"
      end
    end
  end
end
```

- Установить
  - httpd
  - dhcp-server
  - tftp-server


/etc/dhcp/dhcpd.conf:
```
option space pxelinux;
option pxelinux.magic code 208 = string;
option pxelinux.configfile code 209 = text;
option pxelinux.pathprefix code 210 = text;
option pxelinux.reboottime code 211 = unsigned integer 32;
option architecture-type code 93 = unsigned integer 16;

subnet 10.0.0.0 netmask 255.255.255.0 {
        range 10.0.0.100 10.0.0.120;

        class "pxeclients" {
          match if substring (option vendor-class-identifier, 0, 9) = "PXEClient";
          next-server 10.0.0.20;

          if option architecture-type = 00:07 {
            filename "uefi/shim.efi";
            } else {
            filename "pxelinux/pxelinux.0";
          }
        }
}
```

- Скопировать в /var/lib/tftpboot/:
  - pxelinux.0
  - libutil.c32
  - menu.c32
  - libmenu.c32
  - ldlinux.c32
  - vesamenu.c32
  - libcom32.c32
  - chain.c32

*libcom32.c32 и chain.c32 - для возможности загрузиться с локального диска*

/var/lib/tftpboot/pxelinux.cfg/default:
```
default menu.c32
prompt 0
#Время счётчика с обратным отсчётом (установлено 15 секунд)
timeout 150
#Параметр использования локального времени
ONTIME local
#Имя «шапки» нашего меню
menu title OTUS PXE Boot Menu
       #Описание первой строки
       label 1
       #Имя, отображаемое в первой строке
       menu label ^ Graph install CentOS 8.4
       #Адрес ядра, расположенного на TFTP-сервере
       kernel /vmlinuz
       #Адрес файла initrd, расположенного на TFTP-сервере
       initrd /initrd.img
       #Получаем адрес по DHCP и указываем адрес веб-сервера
       append ip=dhcp inst.repo=http://10.0.0.20/centos8
       label 2
       menu label ^ Text install CentOS 8.4
       kernel /vmlinuz
       initrd /initrd.img
       append ip=dhcp inst.repo=http://10.0.0.20/centos8 text
       label 3
       menu label ^ rescue installed system
       kernel /vmlinuz
       initrd /initrd.img
       append ip=dhcp inst.repo=http://10.0.0.20/centos8 rescue
       label 4
       menu label ^ Auto-install CentOS 8 Stream
       #Загрузка данного варианта по умолчанию
       kernel /vmlinuz
       initrd /initrd.img
       append ip=dhcp inst.ks=http://10.0.0.20/config/ks.cfg inst.repo=http://10.0.0.20/centos8/
       label local
       menu label Boot from local drive
       menu default
       kernel chain.c32
       append hd0
```

- Скачать и примонтировать iso-образ CentOS 8 Stream в /mnt/iso
- Скопировать из /mnt/iso/images/pxeboot в /var/lib/tftpboot/
  - initrd.img
  - vmlinuz

/etc/httpd/conf.d/pxeboot.conf:
```
Alias /centos8 /mnt/iso
<Directory /mnt/iso>
    Options Indexes FollowSymLinks
    #Разрешаем подключения со всех ip-адресов
    Require all granted
</Directory>
```

/mnt/config/ks.cfg
```
# Подтверждаем лицензионное соглашение
eula --agreed

# Указываем язык нашей ОС
lang en_US.UTF-8
# Раскладка клавиатуры
keyboard us
# Указываем часовой пояс
timezone UTC+3

# Включаем сетевой интерфейс и получаем ip-адрес по DHCP
network --bootproto=dhcp --device=link --activate
# Задаём hostname otus-c8
network --hostname=otus-c8

# Указываем пароль root пользователя
rootpw vagrant
authconfig --enableshadow --passalgo=sha512
# Создаём пользователя vagrant, добавляем его в группу Wheel
user --groups=wheel --name=vagrant --password=vagrant --gecos="vagrant"

# Включаем SELinux в режиме enforcing
selinux --enforcing
# Выключаем штатный межсетевой экран
firewall --disabled

firstboot --disable

%packages --default
@^minimal
@core
chrony
sudo
mc
%end

# Выбираем установку в режиме командной строки
text
# Указываем адрес, с которого установщик возьмёт недостающие компоненты
url --url="http://10.0.0.20/centos8/BaseOS/"
repo --name=BaseOS --baseurl=https://mirror.yandex.ru/centos/8-stream/BaseOS/x86_64/os/
repo --name=AppStream --baseurl=https://mirror.yandex.ru/centos/8-stream/AppStream/x86_64/os/

# System bootloader configuration
bootloader --location=mbr --append="ipv6.disable=1 crashkernel=auto"

skipx
logging --level=info
zerombr
clearpart --all --initlabel
# Автоматически размечаем диск, создаём LVM
autopart --type=lvm
# Перезагрузка после установки
reboot
```

/etc/httpd/conf.d/configs.conf:
```
Alias /config /mnt/config
<Directory /mnt/config>
    Options Indexes FollowSymLinks
    Require all granted
</Directory>
```

Перезапустить сервисы:

- dhcpd
- httpd

*SELinux не утключен, добавлены правила в firewalld*

### Для проверки запустиь pxeclient - начнётся установка CentOS

```
vagrant up pxeserver
vagrant up pxeclient
```

*Перед стартом cobbler обязательно остановить или уничтожить pxeserver*

```
vagrant destroy pxeserver -f
```

## Задание с * Развернуть стенд с Cobbler

*В данном задании отключены SELinux и firewalld*

Установить
- epel-release
- cobbler
- cobbler-web
- syslinux
- debmirror
- fence-agents

Внести изменения в блок (/etc/cobbler/dhcp.template):
```
subnet 10.0.0.0 netmask 255.255.255.0 {
     #option routers             192.168.1.5;
     #option domain-name-servers 192.168.1.1;
     option subnet-mask         255.255.255.0;
     range dynamic-bootp        10.0.0.100 10.0.0.254;
     default-lease-time         21600;
     max-lease-time             43200;
     next-server                $next_server;
```

В основной конфиг Cobbler внести правки(/etc/cobbler/settings.yaml):
```
next_server: 10.0.0.22
server: 10.0.0.22
manage_tftpd: true
manage_dhcp: true
manage_dns: false
```

Модифицировать реестр, т.к. записи в нём неактуальны:
```json
      "fedora38": {
        "signatures": [
          "Packages"
        ],
        "version_file": "(fedora)-release-38-(.*)\\.noarch\\.rpm",
        "version_file_regex": null,
        "kernel_arch": "kernel-(.*)\\.rpm",
        "kernel_arch_regex": null,
        "supported_arches": [
          "aarch64",
          "ppc64le",
          "s390x",
          "x86_64"
        ],
        "supported_repo_breeds": [
          "rsync",
          "rhn",
          "yum"
        ],
        "kernel_file": "vmlinuz(.*)",
        "initrd_file": "initrd(.*)\\.img",
        "isolinux_ok": false,
        "default_autoinstall": "fedora.ks",
        "kernel_options": "inst.repo=$tree",
        "kernel_options_post": "",
        "boot_files": [],
        "boot_loaders": {}
      }
```

**После правки любых конфигов Cobbler обязательно перезапускать сервис cobblerd**


Создать kickstart для Fedora Server 38 /var/lib/cobbler/templates/fedora.ks:
```
# Generated by Anaconda 38.23.4
# Generated by pykickstart v3.47
#version=F38
# Use graphical install
graphical

# Keyboard layouts
keyboard --vckeymap=ru --xlayouts='us','ru' --switch='grp:alt_shift_toggle'
# System language
lang ru_RU.UTF-8

# Use network installation
url --url=$tree

%packages
@^server-product-environment
@guest-agents
@headless-management

%end

# Run the Setup Agent on first boot
firstboot --enable

# Generated using Blivet version 3.7.1
ignoredisk --only-use=sda
autopart
# Partition clearing information
clearpart --drives=sda --all --initlabel

# System timezone
timezone Europe/Moscow --utc

#Root password
rootpw --lock
user --groups=wheel --name=vagrant --password=$y$j9T$O9sPwRzBPqP/ajnKEKwRKpcF$g.ATyZ9di9VfmjCS1U4Sqvz/.FFU.N3RtPGjdrhlrX0 --iscrypted --gecos="vagrant"
```

Скопировать из /usr/share/syslinux/ в /var/lib/cobbler/loaders/:
- pxelinux.0
- libutil.c32
- menu.c32
- libmenu.c32
- ldlinux.c32
- vesamenu.c32
- libcom32.c32
- chain.c32


Скачать Fedora Server 38 и примонтировать в /mnt/iso

Забиндидь /mnt/distro_mirror на /var/www/cobbler/distro_mirror (связано с ограниченным размером диска)

Сгенерировать boot.0 (grub)
```bash
/usr/share/cobbler/bin/mkgrub.sh
```

Синхронизировать конфигурацию:
```bash
cobbler sync
```

Импорт дистрибутива:
```bash
cobbler import --name=fedora38 --arch=x86_64 --path=/mnt/iso
```

Добавить инсталляцию с именем test:
```bash
cobbler system add --name=test --profile=fedora38-x86_64
cobbler system edit --name=test --interface=eth1 --mac=08:00:27:52:3B:09 --ip-address=10.0.0.100 --netmask=255.255.255.0 --static=1 --dns-name=test.home.su
```

**Выполнить перезагрузку pxeclient и проверить загрузку по сети**

**Playbook:**
```yaml
---
- hosts: pxeserver
  become: true
  gather_facts: false

  tasks:
  - name: Accept login with password from sshd
    ansible.builtin.lineinfile:
      path: /etc/ssh/sshd_config
      regexp: '^PasswordAuthentication no$'
      line: 'PasswordAuthentication yes'
      state: present
    notify:
      - Restart sshd

  - name: Set timezone
    community.general.timezone:
      name: Europe/Moscow

  - name: List all files in directory /etc/yum.repos.d/*.repo
    find:
      paths: "/etc/yum.repos.d/"
      patterns: "*.repo"
    register: repos

  - name: Comment mirrorlist /etc/yum.repos.d/CentOS-*
    ansible.builtin.lineinfile:
      backrefs: true
      path: "{{ item.path }}"
      regexp: '^(mirrorlist=.+)'
      line: '#\1'
    with_items: "{{ repos.files }}"

  - name: Replace baseurl
    ansible.builtin.lineinfile:
      backrefs: true
      path: "{{ item.path }}"
      regexp: '^#baseurl=http:\/\/mirror.centos.org(.+)'
      line: 'baseurl=http://vault.centos.org\1'
    with_items: "{{ repos.files }}"

  - name: Install soft
    ansible.builtin.yum:
      name:
        - vim
        - tcpdump
        - traceroute
        - dhcp-server
        - httpd
        - syslinux-tftpboot.noarch
      state: present
      update_cache: true

  - name: Install TFTPD
    ansible.builtin.yum:
      name:
        - tftp-server
      state: present
    notify:
      - Restart TFTPD

  - name: Copy dhcpd config on pxeserver
    ansible.builtin.copy:
      src: files/pxeserver-dhcpd.conf
      dest: /etc/dhcp/dhcpd.conf
    notify:
      - Restart DHCP daemon

  - name: Copy pxeboot files
    ansible.builtin.copy:
      remote_src: true
      src: "/tftpboot/{{ item }}"
      dest: /var/lib/tftpboot/
    loop:
      - pxelinux.0
      - libutil.c32
      - menu.c32
      - libmenu.c32
      - ldlinux.c32
      - vesamenu.c32
      - libcom32.c32
      - chain.c32

  - name: Create TFTP directory
    ansible.builtin.file:
      path: /var/lib/tftpboot/pxelinux.cfg
      state: directory
      mode: '0755'

  - name: Copy menu config to pxeserver
    ansible.builtin.copy:
      src: files/pxeserver-default
      dest: /var/lib/tftpboot/pxelinux.cfg/default

  - name: Download ISO image
    ansible.builtin.get_url:
      url: http://ftp.mgts.by/pub/CentOS/8-stream/isos/x86_64/CentOS-Stream-8-x86_64-latest-dvd1.iso
      dest: /mnt/
      mode: '0644'

  - name: Mount ISO image
    ansible.posix.mount:
      path: /mnt/iso
      src: /mnt/CentOS-Stream-8-x86_64-latest-dvd1.iso
      fstype: iso9660
      opts: ro,loop
      state: mounted

  - name: Copy initrd and vmlinuz files to TFTP share
    copy:
      src: /mnt/iso/images/pxeboot/{{ item }}
      dest: /var/lib/tftpboot/{{ item }}
      mode: '0755'
      remote_src: true
    with_items:
      - initrd.img
      - vmlinuz


  - name: httpd config
    ansible.builtin.copy:
      src: files/pxeboot.conf
      dest: /etc/httpd/conf.d/pxeboot.conf
      owner: root
      group: root
      mode: 0640
    notify:
      - Restart httpd

  - name: Create config directory
    ansible.builtin.file:
      path: /mnt/config
      state: directory
      mode: '0755'
      setype: httpd_sys_content_t
      seuser: system_u

  - name: Copy ks.cfg
    ansible.builtin.copy:
      src: files/pxeserver-ks.cfg
      dest: /mnt/config/ks.cfg
      owner: root
      group: root
      setype: httpd_sys_content_t
      seuser: system_u
      mode: 0644

  - name: Add configs on httpd
    ansible.builtin.copy:
      src: files/pxeserver-configs.conf
      dest: /etc/httpd/conf.d/configs.conf
      owner: root
      group: root
      mode: 0640
    notify:
      - Restart httpd



  handlers:

  - name: Restart sshd
    ansible.builtin.service:
      name: sshd
      state: reloaded

  - name: Restart TFTPD
    ansible.builtin.service:
      name: tftp
      enabled: true
      state: reloaded
    notify:
      - Accept tftp

  - name: Restart httpd
    ansible.builtin.service:
      name: httpd
      state: reloaded
      enabled: true
    notify:
      - Accept http

  - name: Restart DHCP daemon
    ansible.builtin.service:
      name: dhcpd
      enabled: true
      state: reloaded

  - name: Accept http
    ansible.posix.firewalld:
      service: http
      permanent: true
      state: enabled

  - name: Accept tftp
    ansible.posix.firewalld:
      service: tftp
      permanent: true
      state: enabled



- hosts: cobbler
  become: true
  gather_facts: false

  tasks:

  # Для стенда отключаем
  - name: Disable SELinux
    ansible.posix.selinux:
      state: disabled
    notify:
      - Reboot

  - name: Disable firewalld
    ansible.builtin.service:
      name: firewalld
      state: stopped
      enabled: false

  - name: Accept login with password from sshd
    ansible.builtin.lineinfile:
      path: /etc/ssh/sshd_config
      regexp: '^PasswordAuthentication no$'
      line: 'PasswordAuthentication yes'
      state: present
    notify:
      - Restart sshd

  - name: Set timezone
    community.general.timezone:
      name: Europe/Moscow

  - name: List all files in directory /etc/yum.repos.d/*.repo
    find:
      paths: "/etc/yum.repos.d/"
      patterns: "*.repo"
    register: repos

  - name: Comment mirrorlist /etc/yum.repos.d/CentOS-*
    ansible.builtin.lineinfile:
      backrefs: true
      path: "{{ item.path }}"
      regexp: '^(mirrorlist=.+)'
      line: '#\1'
    with_items: "{{ repos.files }}"

  - name: Replace baseurl
    ansible.builtin.lineinfile:
      backrefs: true
      path: "{{ item.path }}"
      regexp: '^#baseurl=http:\/\/mirror.centos.org(.+)'
      line: 'baseurl=http://vault.centos.org\1'
    with_items: "{{ repos.files }}"

  - name: Install epel-release
    ansible.builtin.yum:
      name:
        - epel-release
      state: present
      update_cache: true

  - name: Install system updates
    ansible.builtin.yum:
      name: '*'
      state: latest
      update_cache: yes

  - name: Install soft
    ansible.builtin.yum:
      name:
        - vim
        - cobbler
        - cobbler-web
        - syslinux
        - debmirror
        - fence-agents
        - pykickstart
        - mc
      state: present

  - name: Comment dists in debmirror
    ansible.builtin.lineinfile:
      backrefs: true
      path: /etc/debmirror.conf
      regexp: '^@dists(.+)'
      line: '#@dists\1'

  - name: Comment arches in debmirror
    ansible.builtin.lineinfile:
      backrefs: true
      path: /etc/debmirror.conf
      regexp: '^@arches(.+)'
      line: '#@arches\1'

  - name: Install TFTPD
    ansible.builtin.yum:
      name:
        - tftp-server
      state: present
    notify:
      - Restart TFTPD

  - name: Install dhcp server
    ansible.builtin.yum:
      name:
        - dhcp-server
      state: present
      update_cache: true

  - name: Copy dhcpd template for cobbler
    ansible.builtin.copy:
      src: files/cobbler-dhcp.template
      dest: /etc/cobbler/dhcp.template

  - name: Copy cobbler settings
    ansible.builtin.copy:
      src: files/cobbler-settings.yaml
      dest: /etc/cobbler/settings.yaml
    notify:
      - Restart Cobbler daemon

  # Обязательно перезапустить сервис, реестр перечитывается только при перезапуске
  - name: Copy distro_signatures.json
    ansible.builtin.copy:
      src: files/cobbler-distro_signatures.json
      dest: /var/lib/cobbler/distro_signatures.json
    notify:
      - Restart Cobbler daemon

  # Автоматическая установка, данные уничтожаются на sda
  - name: Copy fedora-ks.cfg
    ansible.builtin.copy:
      src: files/fedora-ks.cfg
      dest: /var/lib/cobbler/templates/fedora.ks

  - name: Copy from syslinux package files
    ansible.builtin.copy:
      remote_src: true
      src: "/usr/share/syslinux/{{ item }}"
      dest: /var/lib/cobbler/loaders/
    loop:
      - pxelinux.0
      - libutil.c32
      - menu.c32
      - libmenu.c32
      - ldlinux.c32
      - vesamenu.c32
      - libcom32.c32
      - chain.c32

  - name: Download Fedora 38 Server ISO image
    ansible.builtin.get_url:
      url: https://mirror.yandex.ru/fedora/linux/releases/38/Server/x86_64/iso/Fedora-Server-dvd-x86_64-38-1.6.iso
      dest: /mnt/
      mode: '0644'

  - name: Mount ISO image Fedora 38
    ansible.posix.mount:
      path: /mnt/iso
      src: /mnt/Fedora-Server-dvd-x86_64-38-1.6.iso
      fstype: iso9660
      opts: ro,loop
      state: mounted

  - name: Mount distro_mirror
    ansible.posix.mount:
      path: /var/www/cobbler/distro_mirror
      src: /mnt/distro_mirror
      opts: bind
      state: mounted
      fstype: none

  - name: Cobbler create boot.0 (grub)
    ansible.builtin.shell: /usr/share/cobbler/bin/mkgrub.sh

  - name: Start Cobbler daemon
    ansible.builtin.service:
      name: cobblerd
      enabled: true
      state: started

  - name: List distro_mirror fedora38
    ansible.builtin.shell: 'ls /var/www/cobbler/distro_mirror/ | grep fedora38'
    ignore_errors: true
    register: contents

  - name: Cobbler import fedora38
    ansible.builtin.command: cobbler import --name=fedora38 --arch=x86_64 --path=/mnt/iso
    when: contents.stdout == ""

    # Следующие таски только для приемра, что можно назначить любую установку любому хосту
  - name: List cobbler distro fedora38
    ansible.builtin.shell: 'cobbler distro list | grep fedora38'
    register: contents
    ignore_errors: true

  - name: Cobbler add system fedora38
    ansible.builtin.command: cobbler system add --name=test --profile=fedora38-x86_64
    when: contents.stdout == ""

  - name: Cobbler edit system fedora38
    ansible.builtin.command: cobbler system edit --name=test --interface=eth1 --mac=08:00:27:52:3B:09 --ip-address=10.0.0.100 --netmask=255.255.255.0 --static=1 --dns-name=test.mydomain.co
    when: contents.stdout == ""

  - name: Cobbler sync
    ansible.builtin.command: cobbler sync

  - name: Enable TFTPD
    ansible.builtin.service:
      name: tftp
      enabled: true
      state: reloaded

  handlers:

  - name: Restart sshd
    ansible.builtin.service:
      name: sshd
      state: reloaded

  - name: Restart TFTPD
    ansible.builtin.service:
      name: tftp
      state: reloaded

  - name: Restart DHCP daemon
    ansible.builtin.service:
      name: dhcpd
      enabled: true
      state: reloaded

  - name: Restart Cobbler daemon
    ansible.builtin.service:
      name: cobblerd
      enabled: true
      state: restarted

  - name: Reboot
    ansible.builtin.reboot:
```

