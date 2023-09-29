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
      mkdir /home/vagrant/ansible
      mkdir /home/vagrant/ansible/openbsd
      mkdir /home/vagrant/ansible/openbsd/templates
      mkdir /home/vagrant/ansible/openbsd/templates/etc
      SHELL
    #node1.vm.provision "file", source: "pf.conf.j2", destination: "/home/vagrant/ansible/openbsd/templates/etc/pf.conf.j2"
    #node1.vm.provision "file", source: "ansible.cfg", destination: "/home/vagrant/.ansible.cfg"
    #node1.vm.provision "file", source: "hosts.yml", destination: "/home/vagrant/ansible/hosts.yml"
    #node1.vm.provision "file", source: "id_rsa", destination: "/home/vagrant/.ssh/id_rsa"
    #node1.vm.provision "file", source: "user_ssh_config.txt", destination: "/home/vagrant/.ssh/config"
    node1.vm.provision "file", source: "pf.conf.j2", destination: "/tmp/pf.conf.j2"
    node1.vm.provision "file", source: "ansible.cfg", destination: "/tmp/.ansible.cfg"
    node1.vm.provision "file", source: "hosts.yml", destination: "/tmp/hosts.yml"
    node1.vm.provision "file", source: "id_rsa", destination: "/tmp/id_rsa"
    node1.vm.provision "file", source: "id_rsa.pub", destination: "/tmp/id_rsa.pub"
    node1.vm.provision "file", source: "user_ssh_config.txt", destination: "/tmp/ssh_config"
    #fix permissions for the injected files.
    node1.vm.provision "shell", inline: <<-SHELL
      mv /tmp/pf.conf.j2 /home/vagrant/ansible/openbsd/templates/etc/pf.conf.j2
      mv /tmp/.ansible.cfg /home/vagrant/.ansible.cfg
      mv /tmp/hosts.yml /home/vagrant/ansible/hosts.yml
      mv /tmp/id_rsa /home/vagrant/.ssh/id_rsa
      mv /tmp/id_rsa.pub /home/vagrant/.ssh/id_rsa.pub
      mv /tmp/ssh_config /home/vagrant/.ssh/config

      chown -R vagrant:vagrant /home/vagrant
      chown 600 /home/vagrant/.ssh/id_rsa

      wget -O /home/vagrant/.vimrc https://raw.githubusercontent.com/rma92/dot-files/master/linux/.vimrc
      chown vagrant:vagrant /home/vagrant/.vimrc
      SHELL
    node1.vm.provision "shell" do |s|
      ssh_pub_key = File.readlines("id_rsa.pub").first.strip
      s.inline = <<-SHELL
      echo #{ssh_pub_key} >> /root/.ssh/authorized_keys
      echo #{ssh_pub_key} >> /home/vagrant/.ssh/authorized_keys
      pkg_add -I vim-9.0.0192-no_x11
      pkg_add -I git ansible py3-pip
      pip install passlib

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
  end

  config.vm.define "openbsd2" do |node2|
    node2.vm.box = "generic/openbsd7"
    node2.vm.hostname = "openbsd2"
    node2.vm.network "private_network", ip: "192.168.33.20"
    node2.vm.network "public_network", type: "dhcp"
    node2.vm.provider "virtualbox" do |vb|
      vb.memory = "768"
    end
    node2.vm.provision "file", source: "id_rsa", destination: "/tmp/id_rsa"
    node2.vm.provision "file", source: "id_rsa.pub", destination: "/tmp/id_rsa.pub"
    node2.vm.provision "shell", inline: <<-SHELL
      mv /tmp/id_rsa /root/.ssh/id_rsa
      mv /tmp/id_rsa.pub /root/.ssh/id_rsa
      chmod 600 /root/.ssh/id_rsa
      cat /root/.ssh/id_rsa >> /root/.ssh/authorized_keys
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
    node3.vm.provision "file", source: "id_rsa", destination: "/tmp/id_rsa"
    node3.vm.provision "file", source: "id_rsa.pub", destination: "/tmp/id_rsa.pub"
    node3.vm.provision "shell", inline: <<-SHELL
      mv /tmp/id_rsa /root/.ssh/id_rsa
      mv /tmp/id_rsa.pub /root/.ssh/id_rsa
      chmod 600 /root/.ssh/id_rsa
      cat /root/.ssh/id_rsa >> /root/.ssh/authorized_keys
      pkg_add -I vim-9.0.0192-no_x11 git ansible py3-pip
      rcctl stop resolvd
      rcctl disable resolvd
      echo "nameserver 192.168.33.10" > /etc/resolv.conf
      sh /etc/netstart
    SHELL
  end
end
