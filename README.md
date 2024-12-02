## Ansible Host deployment lab

A desire for this project is to make it faster and easier host preparation process using ansible playbooks. This way we are able to personalize machine setup for our purpose. Configuration details are  included in Table of Contents.
I've been using roles to increase quality and readablility. 

## Table of Contents

- [Ansible Host deployment lab](#project-name)
  - [Table of Contents](#table-of-contents)
  - [Introduction](#introduction)
  - [Getting Started](#getting-started)
    - [Prerequisites](#prerequisites)
    - [Installation](#installation)
  - [Playbooks-collection](#Playbooks-collection)
  - [Contributing](#contributing)
  - [License](#license)
  - [Acknowledgments](#acknowledgments)

## Introduction

I don't want to make this post an introduction to Ansible. There are many high-quality sources one can learn about Ansible from. I will present just some concepts that are necessary to know in order to understand my approach.

Ansible is an open-source automation tool that provides a simple and efficient way to automate IT infrastructure tasks. It uses a declarative language called YAML to describe the desired state of systems and performs actions to ensure that the systems match that state.

This is different from shell scripts in many ways:

Ansible uses a declarative approach, where you define the desired state of the system and let Ansible handle the execution details. Shell scripts, on the other hand, are imperative, where you explicitly list out the steps to be executed.
Ansible ensures idempotent execution, which means the same playbook can be run multiple times without causing unintended side effects. Ansible checks the current state of the system and only makes necessary changes to achieve the desired state. Shell scripts often require manual checks and conditions to avoid unintended changes or duplicate actions.
Ansible uses YAML, a human-readable and easy-to-write format, for defining playbooks and tasks. This makes Ansible playbooks more readable and maintainable compared to shell scripts, which often involve more complex syntax.
This makes Ansible more convenient to use instead of shell scripts, if that's only possible. Another thing is that Ansible is push-based. While we'd run shell scripts directly on remote host, with Ansible we "push" the desired configuration from our local machine.

I've organized my playbooks as Ansible Roles. This allows me to self-contain each of my tasks. This was very useful during experimenting phase when doing my home server, because if some role malfunctioned and I had problems with fixing it, or even when I changed my thought, all that was needed from me was to comment out particular role from list of main roles to execute.

Roles, which still need polishing or I'm experimenting with, I can run from separate playbook, and have my battle-tested main one unaffected.


## Getting Started

Explain how to get started with your project. Include step-by-step instructions on setting up and running your project locally.

### Prerequisites
Ansible - To submit playbooks you need ansible package installed on your main host and enable ssh connection between control and managed hosts. It no sense to reinventing wheel once again, clear instructions step by step are posted on official ansible documentation
https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html

Rhel system roles - to simplyfy file system operation its required to apply rhel system roles package 
<br>  *`dnf -y install rhel-system-roles`* <br>
Community general - to use read_csv you must install community.general lib
<br> `ansible-galaxy install community.general`

Provide installation instructions. This may include commands, scripts, or any other necessary steps to set up your project.

## Playbooks-collection

Code part, all plays with description what it does and how it interacts with host machines

This is main part of playbook when roles are induced. Into next points, each of role will be described including ansible code.
```
roles: 
    - role: storage
    - role: groups 
    - role: users
    - role: sshd_config # increase security of sshd service
    - role: motd # setup MOTD banner
    - role: sudo # setup sudo config
    - role: home_backup # home directories backup
    - role: install_software # install all required package
    - role: default_target # set default target
    - role: zsh-plugins # configure oh-my-zsh plugins
    - role: docker-infra # setup docker infrastructure
    - role: ufw # configure UFW firewall
```

    
  # 0. Format and mount additional file systems
    
  We need to add mount points for webserver data, backup and swap.
    Redhat ansible library provides roles to automate filesystem creation by LVM.
    
   IMPORTANT Disk used for lvm ansible config must be cleared without any data or even partition table
    In some cases fdisk cant handle this task properly, but this commands works more effective
    sudo sgdisk -Z /dev/sdx
    source : https://unix.stackexchange.com/questions/202749/remove-all-traces-of-gpt-disk-label
    
   Its require to install rhel roles
    dnf -y install rhel-system-roles
    
    ```
    - name: configure storage for backup, webserver data & swap
      hosts: webservers
      become: true
      
      roles: 
        - name: redhat.rhel_system_roles.storage
	#its one of the roles provided by rhel-system-roles package
	  storage_pools:
	    - name: volume_group1
	      type: lvm
	      disks:
	       - /dev/sdb
	# for different system disk may be named different, its good to precheck it with lsblk cmd
	      volumes:
	        - name: lvolume1
		  size: 512m
		  mount_point: "/webserver_data"
		  fs_type: xfs
		  state: present
		  
		- name: lvolume2
		  size: 1024m
		  mount_point: "/backup"
		  fs_type: xfs
		  state: present
		  
		- name: lvolume_swap
		  size: 1024m
		  fs_type: swap
	 # its not typo :) there is no mount point for swap partition
	          state: present
    ```
    
    
 # 1. Group creation 
    
  First task is group creation, to manage your infrastracture well, we need to provide appropriate groups and permissions
  Fortunately, Ansible ships with more than a thousand modules, including one for reading CSV files. 
  This module can provide exactly whats needful for this task. You will see the module in action below
    
 to use read_csv you must install community.general lib
 ansible-galaxy install community.general

```
- name: create groups from csv
  hosts: all
  tasks:
    - name: reading the csv file before final run
      community.general.read_csv:
        path: groups-list.csv
      register: groups_list
      delegate_to: localhost  # playbook will refer to ansible server files - csv files exists only at server side

    - name: create groups from csv file
      ansible.builtin.group:
        name: "{{ item.Group_name }}"
        gid: "{{ item.GID }}"
        #comment: "This is a Admin group for system management tasks"
        #comments are not available for groups, in opposite to users
        state: present
      loop: "{{ groups_list.list }}"
       # .list notation to retrive record from dictionary
       read_csv format csv to dict - simmilar to python read_csv()
      become: true
       # escalate to root - must have for actual ansible user`

```

 # Here is sample csv file 
 
      Username,uid,First_name,Last_name,Groups,Password
      ansible_dev,9012,Rachel,Booker,Admins,TempPasswd321
      dev1,9013,Michael,Cortney,Admins,Passw000rd333
      web_dev,9015,Kristian,Michalak,Webmasters,Passxdd2137
      jenkins_dev,9014,Donovan,Valenrod,Admins,Passw333Temp4
      ansible_viewer,9016,No,Name,Admins,Tempopassowrd31
 

 # 2. Users creation
 
  Simmilar task connected with first one is user creation we are going to use same python lib from previous tasks.
  lets create a csv file that includes list of users

 to use read_csv you must install community.general lib
 ansible-galaxy install community.general

```
- name: create users from csv
  hosts: all
  tasks:
    - name: reading the csv file before final run
      community.general.read_csv:
        path: users-list.csv
      register: users_list
      delegate_to: localhost

    - name: create users from csv file
      ansible.builtin.user:
        name: "{{ item.Username }}"
        uid: "{{ item.UID }}"
        groups: "{{ item.Groups}}"
        append: true
        password: "{{ item.Password | password_hash('sha512') }}"
        comment: "{{ item.First_name }} {{item.Last_name }}"
        state: present
      loop: "{{ users_list.list }}"
      # .list notation to retrive record from dictionary
      # read_csv format csv to dict - simmilar to python read_csv()
      become: true
      # escalate to root - must have for actual ansible user
```

2.2 In some cases python library that contains read_csv  may produce some errors. 
    There is alternative way to manage users adding process – by variables.
    At first we need to create a file and declare a list of users.
    Example below

users:
  - username: dev1
    groups: Admins
  - username: dev2
    groups: Admins
  - username: dev3
    groups: Admins
  - username: dev4
    groups: Admins
  - username: dev5
    groups: Admins
    
Afterwards, lets create playbook that loop through this list
```      
---
- name: Create multiple local users
  hosts: webservers
  vars_files:
    - vars/users_vars.yml
    
  tasks:
    - name: Add webadmin group
      ansible.builtin.group:
        name: webadmin
        state: present

    - name: Create user accounts
      ansible.builtin.user:
        name: "{{ item['username'] }}"
        groups: "{{ item['groups'] }}"
      loop: "{{ users }}"
   ```   
  # 4. Sudo file settings
  
  Our goal is to add Admins group to sudoers file and verify configuration afterwards. 
  The following example demonstrates how to use the ansible.builtin.lineinfile module to provide a group with sudo access to the root account,
  without prompting the group members for a password

```    
---
- name: modify sudo to allow Admins to sudo
  hosts: all
  become: true
  tasks:
    - name: edit sudoers file by lineinfile module
      path: /etc/sudoers.d/Admins # line depends on linux distro
      #this one tested for Rhel 7 and upper and Centos 7 and upper
      state: present
      create: true # if file dont exists this line will create one
      mode: 0440
      line: "%Admins ALL=(ALL) NO PASSWD: ALL"
      validate: /usr/sbin/visudo -cf %s
      #validate parameter make final check of user sudo access
```
      

  
 # 5.1	Known hosts
  
The following play is an example that uses some advanced techniques to construct an /etc/ssh/ssh_known_hostsfile for all 
managed hosts in the inventory. There might be more efficient ways to accomplish this, because it runs a nested loop on all managed hosts.
It uses the ansible.builtin.slurp module to get the content of the RSA and Ed25519 SSH public host keys in Base64 format,
and then processes the values of the registered variable with the b64decode and trim filters to convert those values back to plain text.

```
	- name: Configure /etc/ssh/ssh_known_hosts files
	  hosts: all
	  tasks:
	    - name: Collect RSA keys
	      ansible.builtin.slurp:
	        src: /etc/ssh/ssh_host_rsa_key.pub
        register: rsa_host_keys
	
	    - name: Collect Ed25519 keys
	      ansible.builtin.slurp:
	        src: /etc/ssh/ssh_host_ed25519_key.pub
	      register: ed25519_host_keys
	
	    - name: Deploy known_hosts
	      ansible.builtin.known_hosts:
	        path: /etc/ssh/ssh_known_hosts
	        name: "{{ item[0] }}"  
	        key: "{{ hostvars[ item[0] ]['ansible_facts']['fqdn'] }} {{ hostvars[ item[0] ][ item[1] ]['content'] | b64decode | trim }}"  
	        state: present
          with_nested:
	        - "{{ ansible_play_hosts }}"  
	        - [ 'rsa_host_keys', 'ed25519_host_keys' ]
```
          
# 5.2 SSH loggin settings
 
 Linux & UNIX machines are mainly managed by ssh protocol therefore its important to harden those connection properly.
 SSH requirements for managed hosts 
 - allow only specified users to connect by ssh
 - disable root login
 - disable password login
 - define maximum amount of connection attempts
 - configure not-default port
 
 After playbook make changes to sshd_conf file, sshd service will be restarted by handler.
 Without restart changes will not  be applied. 

``` 
 handlers:
    - name: "Reload SSHD"
      service:
        name: sshd
        state: restarted
 
 vars_files:
   - vars/ssh_variables
 
 tasks:
  - name: template render
    ansible.builtin.template:
      src: /tmp/j2-template.j2
      dest: /tmp/dest-config-file.txt
```
j2-template.j2 file
```
 # {{ ansible_managed }}
# DO NOT MAKE LOCAL MODIFICATIONS TO THIS FILE BECAUSE THEY WILL BE LOST

Port {{ ssh_port }}
# custom ssh port - defined into ssh_variables file as 2022
ListenAddress {{ ansible_facts['default_ipv4']['address'] }}

HostKey /etc/ssh/ssh_host_rsa_key
HostKey /etc/ssh/ssh_host_ecdsa_key
HostKey /etc/ssh/ssh_host_ed25519_key

SyslogFacility AUTHPRIV

#PermitRootLogin {{ root_allowed }}
AllowGroups {{ groups_allowed }}

AllowUsers {{ users_allowed }}
# only ansible and dev1 users are allowed to open ssh connections with managed hosts

AuthorizedKeysFile /etc/.rht_authorized_keys .ssh/authorized_keys

#PasswordAuthentication {{ passwords_allowed }}

ChallengeResponseAuthentication no

GSSAPIAuthentication yes
GSSAPICleanupCredentials no
MaxSessions 5 # no more than 5 session allowed to establish ssh connection by 1 ip address
UsePAM yes

X11Forwarding yes
UsePrivilegeSeparation sandbox

AcceptEnv LANG LC_CTYPE LC_NUMERIC LC_TIME LC_COLLATE LC_MONETARY LC_MESSAGES
AcceptEnv LC_PAPER LC_NAME LC_ADDRESS LC_TELEPHONE LC_MEASUREMENT
AcceptEnv LC_IDENTIFICATION LC_ALL LANGUAGE
AcceptEnv XMODIFIERS

Subsystem sftp	/usr/libexec/openssh/sftp-server
```

# 6. MOTD 
This playbook edit /etc/motd file thats sets welcome message that displays immediately  after user log in.
During the loggin user see hostname, ipaddress, destro version and memory info(used/available) swap and RAM

 It copies the template motd.j2 and change variable with ansible_facts of managed hosts

```
- name: change motd
  hosts: all
  remote_user: devops
  become: true
  vars:
    - system_owner: damianus@serwis.com
  tasks:
    - name: configure MOTD
      ansible.builtin.template:
      src: motd.j2
      dest: /etc/motd
      owner: root
      group: root
      mode: 0644
```
EXAMPLE OF TEMPLATE - jinja2 template
```
THis is a system {{ ansible_facts['fqdn'] }}
This is a {{ ansible_facts['distribution' }} version {{ ansible_facts['distribution_version'] }} system
ansible_facts['memtotal_mb'] and ansible_facts['processor_count']
memory used ansible_facts['memory_mb']['nocache']['used'] / free memory ansible_facts['memory_mb']['nocache']['free']
used swap ansible_facts['memory_mb']['swap']['used'] / free swap ansible_facts['memory_mb']['free']
```

# 7. Backups

Playbook creates and sets directory for files storage. In this example we create synchornize copy of home directories.
IMPORTANT: its not good playbook for deployment procedure, that role wont be implement during machine creation.
We are going to use that script to perform cyclical backup of user files.

Usually cron jobs are used for this task but I changed scheduled script for synchronize ansible module
•	synchronize is a wrapper around rsync to make common tasks in your playbooks quick and easy.
•	It is run and originates on the local host where Ansible is being run.
•	Of course, you could just use the command action to call rsync yourself, 
but you also have to add a fair number of boilerplate options and host facts.

 To install it, use - ansible-galaxy collection install ansible.posix
```
---
- name: using synchronize module
  hosts: all
  become: true
  tasks:
    - name: synchronize and backup home dirs for all users
      ansible.posix.synchronize:
        src: /home
        dest: /backup-local2/backup/home
        recursive: no # enables an archive backup
	
OPTIONAL - use cron jobs to automate backups. It runs  backup-home script at 24:00 every Saturday

- name: Schedule backups for my home directory
  ansible.builtin.cron:
    name: Backup my home directory
    user: ansible
    job: /scripts/backup-home.sh
    minute: 0
    hour: 24
    weekday: 6
```    
  
8. Install required software - Work in progress

# 9. Set default target  multi-user.target

Purpose of this play is to make sure all managed host are booted from multi-user.target
If someone is not familiar with targets, its upgraded more elastic version of inits.

```
---
- name: Make sure default systemd target is multi-user.target
  hosts: all
  gather_facs: false
  #prevent ansible collecting facts info from managed hosts
  vars:
    selected_target: "multi-user.target"
    
  tasks:
    - name: check currently used systemd target
      ansible.builtin.command:
        cmd: systemctl get-default
      changed_when: false
      # its gathering information so task should never report changed state
      register: target
      # IMPORTANT - it saves output of cmd command to target variable
      
   - name: set default systemd target
     ansible.builtin.command:
       cmd: systemctl set-default {{ selected_target }}
     when: selected_target not in target['stdout']
     # change affects only in case target is different than multi-user-target
     become: true
     # root permission is only needful for this task, so its better to enable it just for 1 task
```

## Contributing

Explain how others can contribute to your project. Include guidelines for submitting bug reports, feature requests, or pull requests. 

## License

This project is licensed under the [License Name] License - see the [LICENSE.md](LICENSE.md) file for details.

## Acknowledgments

Give credit to any third-party libraries, frameworks, or tools that you used in your project. You can also acknowledge contributors or sources of inspiration.


