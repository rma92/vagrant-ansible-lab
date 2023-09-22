Vagrant.configure("2") do |config|
  config.vm.define "openbsd1" do |node1|
    node1.vm.box = "generic/openbsd7"
    node1.vm.hostname = "openbsd1"
    node1.vm.network "private_network", ip: "192.168.33.10"
    node1.vm.network "public_network", type: "dhcp"
    node1.vm.provider "virtualbox" do |vb|
      vb.memory = "512"
    end
    node1.vm.provision "shell", inline: <<-SHELL
      pkg_add -I vim-gtk3 git
    SHELL
  end

  config.vm.define "openbsd2" do |node2|
    node2.vm.box = "generic/openbsd7"
    node2.vm.hostname = "openbsd2"
    node2.vm.network "private_network", ip: "192.168.33.20"
    node2.vm.network "public_network", type: "dhcp"
    node2.vm.provider "virtualbox" do |vb|
      vb.memory = "512"
    end
    node2.vm.provision "shell", inline: <<-SHELL
      pkg_add -I vim-gtk3 git
    SHELL
  end
end
