Vagrant.configure("2") do |config|
  config.vm.define "openbsd1" do |node1|
    node1.vm.box = "generic/openbsd7"
    node1.vm.hostname = "openbsd1"
    node1.vm.network "private_network", ip: "192.168.33.10"
  end

  config.vm.define "openbsd2" do |node2|
    node2.vm.box = "generic/openbsd7"
    node2.vm.hostname = "openbsd2"
    node2.vm.network "private_network", ip: "192.168.33.20"
  end
end
