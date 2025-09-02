Vagrant.configure("2") do |config|
  config.ssh.insert_key = true
  config.vm.box = "ubuntu/jammy64"

  (1..2).each do |i|
    config.vm.define "web#{i}" do |node|
      node.vm.hostname = "web#{i}"
      node.vm.network "private_network", ip: "192.168.56.1#{i}"
      node.vm.provider "virtualbox" do |vb|
        vb.memory = 1024
        vb.cpus   = 1
      end
      node.vm.provision "shell", inline: <<-SHELL
        apt-get update -y
        apt-get install -y python3 python3-apt curl
      SHELL
    end
  end
end
