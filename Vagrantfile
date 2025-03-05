Vagrant.configure("2") do |config|
  # Base VM OS configuration.
  config.vm.box = "debian/bookworm64"
#  config.vm.box = "ubuntu/jammy64"
  config.vm.provider :virtualbox do |v|
    v.memory = 1512
    v.cpus = 2
  end

  # Define three VMs with static private IP addresses.
  boxes = [
    { :name => "node1", :ip => "192.168.57.11" },
    { :name => "node2", :ip => "192.168.57.12" },
    { :name => "barman", :ip => "192.168.57.13" }
  ]
  
  # Provision each of the VMs.
  boxes.each do |opts|
    config.vm.define opts[:name] do |config|
      config.vm.hostname = opts[:name]
      config.vm.network "private_network", ip: opts[:ip]

      # Provision with Ansible for each machine
      if opts[:name] == "barman"
        config.vm.provision "ansible" do |ansible|
          ansible.playbook = "ansible/provision.yml"	# Путь к корневому playbook
          ansible.config_file = "ansible/ansible.cfg"	# Путь к конфиг файлу ansible
          ansible.become = true				# Если нужны привилегии суперпользователя
          ansible.limit = "all"				# Чтобы был доступ ко всем ВМ
#          ansible.inventory_path = "ansible/hosts.ini"
#          ansible.host_key_checking = "false"
          # Add remote_tmp configuration
#          ansible.extra_vars = {
#            'ansible_remote_tmp' => '/tmp/.ansible-tmp'
#          }
        end
      end
    end
  end
end
