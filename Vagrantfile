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
