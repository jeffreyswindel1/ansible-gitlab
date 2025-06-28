# -*- mode: ruby -*-
# vi: set ft=ruby :
require 'net/http'
require 'uri'
require 'yaml'

VAGRANT_COMMAND = ARGV[0]

ansible_vars = YAML.load_file("./ansible/vars/vars.yml")

box = "bento/ubuntu-24.04"
box_default_username = "vagrant"
provider = "virtualbox"
hostname = ansible_vars["compose_project_name"]
python_version = "python3.12"
ansible_verbose_level = ENV.fetch("VAGRANT_ANSIBLE_VERBOSE_LEVEL") { "v" }
forward_ports = ENV.fetch("VAGRANT_FORWARD_PORTS") { "true" }
personal_key = ENV.fetch("VAGRANT_PERSONAL_SSH_KEY") { "false" }
home_dir = ENV["HOME"]

# Convert string to boolean
def convert_to_boolean(string)
    string.casecmp("true").zero?
end

if convert_to_boolean("#{personal_key}")
    ssh_key = ENV.fetch("VAGRANT_SSH_KEY") { "id_ed25519" }
    private_key_path = "#{home_dir}/.ssh/#{ssh_key}"
    ssh_pub_key = File.readlines("#{private_key_path}.pub").first.strip
else
    private_key_path = "#{home_dir}/.vagrant.d/insecure_private_keys/vagrant.key.ed25519"
    # Get public key from file
    insecure_key = "https://raw.githubusercontent.com/hashicorp/vagrant/refs/heads/main/keys/vagrant.pub"
    ssh_pub_key = Net::HTTP.get(URI.parse("#{insecure_key}"))
end

Vagrant.configure("2") do |config|
    vm_box = "#{box}"

    config.vm.box_download_insecure = true
    config.vm.provider "virtualbox" do |v|
        v.memory = 4096
        v.cpus = 2
        v.customize [ "guestproperty", "set", :id, "/VirtualBox/GuestAdd/VBoxService/--timesync-set-threshold", 1000 ]
        v.customize [ "modifyvm", :id, "--cpuexecutioncap", "50" ]
        v.customize [ "modifyvm", :id, "--nested-hw-virt", "on" ]
    end

    config.ssh.insert_key = false
    config.ssh.verify_host_key = false

    username = "#{box_default_username}"
    
    if VAGRANT_COMMAND == "ssh"
        username = ansible_vars["users"][0]["username"]
    end

    config.ssh.username = "#{username}"
    config.ssh.private_key_path = ["#{private_key_path}"]

    if convert_to_boolean("#{forward_ports}")
        ports = [{guest: 80, host: 8080, protocol: "tcp"},{guest: 443, host: 8443, protocol: "tcp"},{guest: 22, host: 2222, protocol: "tcp"}]
        ports.each do |port|
            config.vm.network "forwarded_port", guest: port[:guest], host: port[:host], protocol: port[:protocol]
        end
    end

    config.vm.define "#{hostname}" do |c|
        c.vm.box = vm_box
        c.vm.hostname = "#{hostname}"
        c.vm.provision "shell", inline: <<-EOC
            systemctl is-active --quiet sshd.service
            if [ $? == 0 ]; then
                sudo sed -i -e "\\#PasswordAuthentication yes# s#PasswordAuthentication no#g" /etc/ssh/sshd_config
                sudo systemctl restart sshd.service
            fi
            printf "finished initial shell config\n"
        EOC
        c.vm.provision "ansible", run: "always" do |ansible|
            ansible.compatibility_mode = "2.0"
            ansible.playbook    = "./ansible/playbook.yml"
            ansible.extra_vars  = { ansible_python_interpreter: "/usr/bin/python3", ssh_pub_key: "#{ssh_pub_key}" }
            ansible.verbose     = "#{ansible_verbose_level}"
        end
    end
end
