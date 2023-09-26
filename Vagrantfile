Vagrant.configure("2") do |config|
config.vm.define "openbsd1" do |node1|
  node1.vm.box = "generic/openbsd7"
  node1.vm.hostname = "openbsd1.example.gs"
  node1.vm.network "private_network", ip: "192.168.33.10"
  node1.vm.network "public_network", type: "dhcp"
  node1.vm.provider "virtualbox" do |vb|
    vb.memory = "768"
  end
  node1.vm.provision "shell", inline: <<-SHELL
    pkg_add -I vim-9.0.0192-no_x11 git ansible py3-pip

    # Define the unbound configuration
    unbound_conf='
    server:
        interface: 0.0.0.0
        interface: ::0
        port: 53
        access-control: 0.0.0.0/0 allow

    # local-zone: "google.com." always_nxdomain
    local-zone: "localhost." redirect
    local-zone: "example.gs." transparent
    local-data: "openbsd1.example.gs. IN A 192.168.33.10"
    local-data: "openbsd2.example.gs. IN A 192.168.33.20"
    local-data: "openbsd3.example.gs. IN A 192.168.33.30"

    forward-zone:
        name: "."
        forward-addr: 8.8.8.8
    '

    # Write the unbound configuration to /var/unbound/etc/unbound.conf
    echo "$unbound_conf" | tee /var/unbound/etc/unbound.conf > /dev/null
    rcctl enable unbound
    rcctl start unbound
    rcctl stop resolvd
    rcctl disable resolvd
    echo "search example.gs
    nameserver 127.0.0.1" > /etc/resolv.conf
    sh /etc/netstart
  SHELL
end


  config.vm.define "openbsd2" do |node2|
    node2.vm.box = "generic/openbsd7"
    node2.vm.hostname = "openbsd2"
    node2.vm.network "private_network", ip: "192.168.33.20"
    node2.vm.network "public_network", type: "dhcp"
    node2.vm.provider "virtualbox" do |vb|
      vb.memory = "768"
    end
    node2.vm.provision "shell", inline: <<-SHELL
      pkg_add -I vim-9.0.0192-no_x11 git ansible py3-pip
      rcctl stop resolvd
      rcctl disable resolvd
      echo "nameserver 192.168.33.10" > /etc/resolv.conf
      sh /etc/netstart
    SHELL
  end

  config.vm.define "openbsd3" do |node3|
    node3.vm.box = "generic/openbsd7"
    node3.vm.hostname = "openbsd3"
    node3.vm.network "private_network", ip: "192.168.33.30"
    node3.vm.network "public_network", type: "dhcp"
    node3.vm.provider "virtualbox" do |vb|
      vb.memory = "768"
    end
    node3.vm.provision "shell", inline: <<-SHELL
      pkg_add -I vim-9.0.0192-no_x11 git ansible py3-pip
      rcctl stop resolvd
      rcctl disable resolvd
      echo "nameserver 192.168.33.10" > /etc/resolv.conf
      sh /etc/netstart
    SHELL
  end
end
