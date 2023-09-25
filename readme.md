# OpenBSD test Vagrant environment
Two OpenBSD VMs.

Cache the box locally (not required, only if you haven't used it and want to start offline):
```
vagrant box add generic/openbsd7
```
Start up:
```
vagrant up --paralell
```

SSH to a VM:
```
vagrant ssh <vmname>
```

Get rid of everything:
```
vagrant destroy --force
```

# Setting up the ansible lab
We will use openbsd1 as the control node.

SSH to Openbsd1 to set things up.
```
sudo pkg_add ansible
wget -O ~/.vimrc https://raw.githubusercontent.com/rma92/dot-files/master/linux/.vimrc
vim ~/.ansible.cfg
```

Put the following basic config in the .ansible.cfg file:
```
[defaults]
inventory = /home/vagrant/ansible/hosts.yml
interpreter_python = auto
```

Create the directory to store `hosts.yml`.

```
mkdir ~/ansible
cd ~/ansible
```

Make a hosts.yml
```
all:
  vars:
    ansible_python_interpreter: /usr/local/bin/python3
    ansible_become: yes
    ansible_become_method: sudo
  children:
    vagrant_env:
      openbsd1:
        user: user
        user_passwd: "{{ host_user_passwd }}"
        root_passwd: "{{ host_root_passwd }}"
        ansible_become_pass: "{{ host_user_passwd }}"
        ansible_python_interpreter: /usr/local/bin/python3
```

Using the Ansible Vault - Create a directory and a passwd.yml file.
```
mkdir -p ~/ansible/openbsd
cd ~/ansible/openbsd
ansible-vault create passwd.yml
```
Set a vault password, add the following content:
```
host_user_passwd: ELqZ9L70SSOTjnE0Jq
host_root_passwd: tgM2Q5h8WCeibIdJtd
```
(Generate your own passwords)

View the vault:
```
ansible-vault view passwd.yml
```
Edit the vault:
```
ansible-vault edit passwd.yml
```


