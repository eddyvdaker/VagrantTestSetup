# -*- mode: ruby -*-
# vi: set ft=ruby

require 'fileutils'

VAGRANT_ROOT                = File.dirname(File.expand_path(__FILE__))
VAGRANT_DISKS_DIRECTORY     = File.join(VAGRANT_ROOT, "disks")
VAGRANT_CONTROLLER_NAME     = "Virtual I/O Device SCSI controller"
VAGRANT_CONTROLLER_TYPE     = "virtio-scsi"

local_disks = [
    { :filename => "disk2", :size => 50, :port => 5},
    { :filename => "disk3", :size => 50, :port => 6}
]

Vagrant.configure("2") do |config|
    config.vm.box = "ubuntu/focal64"

    # Set configuration for VirtualBox VM resources
    config.vm.provider "virtualbox" do |vb|
        vb.gui = false
        vb.cpus = 2
        vb.memory = 4096

        vb.customize ["modifyvm", :id, "--vram", "128"]
    end

    # Setup NAT network
    config.vm.network "private_network", type: "dhcp"
    config.vm.network "forwarded_port", guest: 22, host: 50022
    config.vm.network "forwarded_port", guest: 80, host: 50080
    config.vm.network "forwarded_port", guest: 443, host: 50443

    # Setup bridged network
    config.vm.network "public_network", :mac => "080027B69C01", ip: "192.168.1.175"

    # Create disks before "up" action
    config.trigger.before :up do |trigger|
        trigger.name = "Create disks"
        trigger.ruby do
            unless File.directory?(VAGRANT_DISKS_DIRECTORY)
                FileUtils.mkdir_p(VAGRANT_DISKS_DIRECTORY)
            end
            local_disks.each do |local_disk|
                local_disk_filename = File.join(VAGRANT_DISKS_DIRECTORY, "#{local_disk[:filename]}.vdi")
                unless File.exists?(local_disk_filename)
                    puts "Create \"#{local_disk[:filename]}\" disk"
                    system("vboxmanage createmedium --filename #{local_disk_filename} --size #{local_disk[:size] * 1024} --format VDI")
                end
            end
        end
    end

    # Create storage controller on first run
    unless File.directory?(VAGRANT_DISKS_DIRECTORY)
        config.vm.provider "virtualbox" do |storage_provider|
            storage_provider.customize ["storagectl", :id, "--name", VAGRANT_CONTROLLER_NAME, "--add", VAGRANT_CONTROLLER_TYPE,
                "--hostiocache", "off"]
        end
    end

    # Attach storage devices
    config.vm.provider "virtualbox" do |storage_provider|
        local_disks.each do |local_disk|
            local_disk_filename = File.join(VAGRANT_DISKS_DIRECTORY, "#{local_disk[:filename]}.vdi")
            unless File.exists?(local_disk_filename)
                storage_provider.customize ["storageattach", :id, "--storagectl", VAGRANT_CONTROLLER_NAME, "--port", local_disk[:port],
                    "--device", 0, "--type", "hdd", "--medium", local_disk_filename]
            end
        end
    end

    # Provision extra disks config to VM
    config.vm.provision "file", source: "./disk1.txt", destination: "./disk1.txt"
    config.vm.provision "file", source: "./disk2.txt", destination: "./disk2.txt"

    # Setup extra disks on VM
    config.vm.provision "shell", inline: <<-SHELL
        devices=$(lsblk | grep 50G | grep -oe "sd[a-z]")
        partition_number=1
        declare -i disk_number=1
        for dev in $devices 
        do
            sfdisk /dev/$dev < disk$disk_number.txt
            mkfs.ext4 /dev/$dev$partition_number
            mkdir /disk-$disk_number
            mount -t ext4 /dev/$dev$partition_number /disk-$disk_number
            echo "/dev/$dev$partition_number /disk-$disk_number ext4 rw,relatime 0 1" >> /etc/fstab
            disk_number=$(( $disk_number + 1))
        done
    SHELL

    # Cleanup after "destroy" action
    config.trigger.after :destroy do |trigger|
        trigger.name = "Cleanup disks directory"
        trigger.ruby do
            local_disks.each do |local_disk|
                local_disk_filename = File.join(VAGRANT_DISKS_DIRECTORY, "#{local_disk[:filename]}.vdi")
                if File.exists?(local_disk_filename)
                    puts "Deleting \"#{local_disk[:filename]}\" disk"
                    system("vboxmanage closemedium disk #{local_disk_filename} --delete")
                end
            end
            sleep(1)        # Sometimes the deletion of the directory triggers before
                            # the disks have been deleted, which causes an error. This delay 
                            # is added to prevent this overlap.
            if File.exists?(VAGRANT_DISKS_DIRECTORY)
                FileUtils.rmdir(VAGRANT_DISKS_DIRECTORY)
            end
        end
    end

    # Add shared folder between host and VM
    unless File.exists?("./_VagrantData")
        FileUtils.mkdir_p("./_VagrantData")
    end
    config.vm.synced_folder "./_VagrantData", "/data"
    config.vm.provision "shell", inline: <<-SHELL
        chmod -R 666 /data
        umask 000 /data
    SHELL

    # Install updates and packages
    config.vm.provision "shell", inline: <<-SHELL
        echo "Updating repos"
        apt-get update
        echo "Upgrading packages"
        apt-get upgrade -y
        echo "Install zsh"
        apt-get install zsh -y
    SHELL

    # Setup additional user
    config.vm.provision "shell", inline: <<-SHELL
        pass=$(perl -e 'print crypt("testpass", "testsalt"), "\n"')
        useradd -m test -s $(which zsh) -p "$pass" -G sudo
        export HOME=/home/test
        sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)" "" --unattended
        chown -R test:test /home/test/.oh-my-zsh
        chown test:test /home/test/.zshrc
        mkdir /home/test/.ssh
        touch /home/test/.ssh/authorized_keys
        chown -R test:test /home/test/.ssh
    SHELL

    # Add host public key to VM authorized_keys
    config.vm.provision "shell" do |s|
        ssh_pub_key = File.readlines("#{Dir.home}/.ssh/id_rsa.pub").first.strip
        s.inline = <<-SHELL
            echo #{ssh_pub_key} >> "/home/vagrant/.ssh/authorized_keys"
            echo #{ssh_pub_key} >> "/home/test/.ssh/authorized_keys"
        SHELL
    end
end