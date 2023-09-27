# OpenBSD test Vagrant environment
Three OpenBSD VMs.

Get going fast:
```
ssh-keygen -t rsa -b 4096 -N "" -f ./id_rsa
vagrant up
```
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
sudo pkg_add ansible py3-pip
pip install passlib
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
vi hosts.yml
```
Put the contents:
```
all:
  vars:
    ansible_python_interpreter: /usr/local/bin/python3
    ansible_become: yes
    ansible_become_method: sudo
  children:
    vagrant_env:
      hosts:
        openbsd2.example.gs:
          user: user
          user_passwd: "{{ host_user_passwd }}"
          root_passwd: "{{ host_root_passwd }}"
          ssh_pub_key: "{{ lookup('file', '~/.ssh/openbsd2_root.pub') }}"
          ansible_become_pass: "{{ host_user_passwd }}"
          ansible_python_interpreter: /usr/local/bin/python3
```

## Set up an SSH Key
Use vagrant ssh to connect to openbsd2, `sudo -s` to become root.  `cd` to root's home directory, and run ssh-keygen.  Leave the defaults.
```
cd ~
ssh-keygen
cd ~/.ssh
echo `cat id_rsa.pub` >> authorized_keys
```

## Set up an SSH config file on openbsd1 so ansible can log into the other servers as root.
Copy the contents of root's RSA key to openbsd1.
```
cat ~/.ssh/id_rsa
```

```
vi ~/.ssh/openbsd2_root
# paste the root key
chmod 600 ~/.ssh/openbsd2_root
vi ~/.ssh/config
```

```
Host *
  AddKeysToAgent yes
  AddressFamily inet

Host openbsd2.example.gs openbsd2
  Hostname openbsd2.example.gs
  Port 22
  User root
  IdentityFile ~/.ssh/openbsd2_root
```

## Set up Ansible Vault
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

## Testing
```
cd ~/ansible/openbsd/
ansible -m ping --ask-vault-pass --extra-vars '@passwd.yml' openbsd2.example.gs -u root
```

## Playbooks
We will make a two playbooks, one for pf, and one for the basic setup.
`~/ansible/openbsd/templates/etc/pf.conf.j2`

```
wan0 = "{{ ansible_default_ipv4.interface }}"
icmp_types = "{ echoreq unreach }"

# pfctl -t "table" -T show
#   Display table contents.
# pfctl -t bruteforce -T expire 86400
#   Remove bruteforce table entries older than 86400 seconds.
# pfctl -t bruteforce -T delete 12.34.56.78
#   Immediately delete bruteforce table entry

table <bruteforce> persist

table <martians> {
  0.0.0.0/8 10.0.0.0/8 100.64.0.0/10            \
  127.0.0.0/8 169.254.0.0/16 172.16.0.0/12      \
  192.0.0.0/24 192.0.2.0/24 192.88.99.0/24      \
  192.168.0.0/16 198.18.0.0/15 198.51.100.0/24  \
  203.0.113.0/24 224.0.0.0/3 255.255.255.255/32 \
  ::/128 ::/96 ::1/128 ::ffff:0:0/96 100::/64   \
  2001:10::/28 2001:2::/48 2001:db8::/32        \
  3ffe::/16 fec0::/10 fc00::/7 }

set block-policy return
set loginterface egress
set skip on lo0

match in all scrub (no-df random-id max-mss 1440)
antispoof quick for { egress $wan0 }
block quick from <bruteforce>
block in quick on egress from <martians> to any
block return out quick on egress from any to <martians>
block all

# SSH
# Add brute force hosts (5 attempts within 3 seconds)
# to the bruteforce table.
pass in log on $wan0 proto tcp to port { {{ ssh_port }} } \
    keep state (max-src-conn 15, max-src-conn-rate 5/3, \
        overload <bruteforce> flush global) label ssh-traffic

pass out quick

# ICMP
pass in on egress inet proto icmp all icmp-type $icmp_types
pass in on egress inet6 proto icmp6 all icmp6-type $icmp_types
pass in on egress inet6 proto icmp6 all \
  icmp6-type { routeradv neighbrsol neighbradv }
```

`~/ansible/openbsd/setup-pb.yml`

```
# Initial server setup
#
---
- hosts: all
  become: true
  vars:
    ssh_port: "22"
    tmzone: America/New_York
    user_shell: /usr/local/bin/bash
    doas: |
      permit persist :wheel
    vimrc: |
      set mouse-=a
    dot_profile: |
      export PATH HOME TERM
      # This is the standard OpenBSD PATH,
      # defined for easy customization.
      PATH=/bin:/sbin:/usr/bin:/usr/sbin
      PATH=$PATH:/usr/X11R6/bin
      PATH=$PATH:/usr/local/bin:/usr/local/sbin
      PATH=$PATH:/usr/games
      PATH=$HOME/bin:$PATH
      # include .bashrc if it exists
      if [ -f "$HOME/.bashrc" ]; then
          . "$HOME/.bashrc"
      fi
      # Set two-line user prompt
      PS1="\e[0;32m\u@\h\e[m:\w\n$ "
    dot_root_profile: |
      # include .kshrc if it exists
      if [ -f "$HOME/.kshrc" ]; then
          . "$HOME/.kshrc"
      fi
    dot_bashrc: |
      alias l='ls -CF'
      alias la='ls -AF'
      alias ll='ls -alF'
    dot_kshrc: |
      alias l='ls -CF'
      alias ll='ls -alF'
  tasks:
    # Update and install the base software
    - name: Apply all available system patches.
      command: syspatch
      register: syspatch
      failed_when: syspatch.rc != 0 and syspatch.rc != 2
      changed_when: syspatch.rc == 0
    - name: Update all packages on the system.
      command: pkg_add -u
    - name: Reboot the server if needed.
      reboot:
        msg: "Reboot initiated by Ansible because of syspatch updates."
        connect_timeout: 5
        reboot_timeout: 600
        pre_reboot_delay: 0
        post_reboot_delay: 15
        test_command: whoami
      when: syspatch.rc == 0
    - name: Install a base set of software packages.
      openbsd_pkg:
        name:
          - bash
          - curl
          - git
          - htop
          - pwgen
          - rsync--
          - vim--no_x11
        state: present
    # Host Setup
    - name: Set static hostname.
      hostname:
        name: "{{ inventory_hostname }}"
    - name: Add IPv4 FQDN to /etc/hosts.
      lineinfile:
        dest: /etc/hosts
        regexp: '^127\.0\.0\.1'
        line: "127.0.0.1 {{ inventory_hostname }}"
        state: present
    - name: Add IPv6 FQDN to /etc/hosts.
      lineinfile:
        dest: /etc/hosts
        regexp: '^\:\:1'
        line: "::1       {{ inventory_hostname }}"
        state: present
    - name: Add FQDN to /etc/myname.
      lineinfile:
        dest: /etc/myname
        regexp: '^(.*)'
        line: "{{ inventory_hostname }}"
        state: present
    - name: Set timezone.
      timezone:
        name: "{{ tmzone }}"
      notify:
        - restart cron
    - name: Set ssh port port number.
      lineinfile:
        dest: /etc/ssh/sshd_config
        regexp: 'Port '
        line: 'Port {{ ssh_port }}'
        state: present
      notify:
        - restart sshd
    - name: Disable root password login via SSH.
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PermitRootLogin'
        line: 'PermitRootLogin prohibit-password'
      notify:
        - restart sshd
    - name: Disable tunneled clear-text passwords.
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PasswordAuthentication'
        line: 'PasswordAuthentication no'
      notify:
        - restart sshd
    - name: Configure /etc/doas.conf.
      copy:
        dest: /etc/doas.conf
        content: "{{ doas }}"
        owner: root
        group: wheel
        mode: 0644
    - name: Create regular user group.
      group:
        name: "{{ user }}"
        state: present
    - name: Create regular user with doas privileges.
      user:
        name: "{{ user }}"
        group: "{{ user }}"
        password: "{{ user_passwd | password_hash('bcrypt') }}"
        groups: wheel
        append: true
        shell: /usr/local/bin/bash
    - name: Ensure authorized keys for remote user is installed.
      authorized_key:
        user: "{{ user }}"
        key: "{{ ssh_pub_key }}"
    - name: Ensure authorized key for root user is installed.
      authorized_key:
        user: root
        key: "{{ ssh_pub_key }}"
    - name: Update root user password.
      user:
        name: root
        password: "{{ root_passwd | password_hash('bcrypt') }}"
    - name: Configure user .vimrc.
      copy:
        dest: /home/{{ user }}/.vimrc
        content: "{{ vimrc }}"
        owner: "{{ user }}"
        group: "{{ user }}"
        mode: 0644
    - name: Configure root .vimrc.
      copy:
        dest: /root/.vimrc
        content: "{{ vimrc }}"
        owner: root
        group: wheel
        mode: 0644
    - name: Configure user .profile,
      copy:
        dest: /home/{{ user }}/.profile
        content: "{{ dot_profile }}"
        owner: "{{ user }}"
        group: "{{ user }}"
        mode: 0644
    - name: Configure user .bashrc,
      copy:
        dest: /home/{{ user }}/.bashrc
        content: "{{ dot_bashrc }}"
        owner: "{{ user }}"
        group: "{{ user }}"
        mode: 0644
    - name: Update root .profile
      blockinfile:
        path: /root/.profile
        content: "{{ dot_root_profile }}"
        owner: root
        group: wheel
        mode: 0644
    - name: Configure root .kshrc.
      copy:
        dest: /root/.kshrc
        content: "{{ dot_kshrc }}"
        owner: root
        group: wheel
        mode: 0644
    - name: Configure pf firewall.
      template:
        src: etc/pf.conf.j2
        dest: /etc/pf.conf
        owner: root
        group: wheel
        mode: 0600
      notify:
        - reload pf.conf
    - meta: end_play
  handlers:
    - name: restart cron
      service:
        name: cron
        state: restarted
    - name: restart sshd
      service:
        name: sshd
        state: restarted
    - name: reload pf.conf
      command: pfctl -f /etc/pf.conf
```

## Run the playbook
```
cd ~/ansible/openbsd
ansible-playbook --ask-vault-pass --extra-vars '@passwd.yml' setup-pb.yml -l openbsd2.example.gs -u root
```
