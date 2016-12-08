# **Installing Sipxcom cluster in DevOps style with Ansible and Vagrant**


The eZuce developers are preparing to move to microservices with a completely new design for our next generation unified collaboration system, we too (ops, support and generally non-dev staff) need to get familiar with the basics of the new DevOps trend.

I will write a few tutorials around Ansible, Vagrant and the ELK stack to help myself and our customers understand how we can use these devops tools to ease sipXcom/Uniteme installation and upgrades. After that I will introduce ELK (Elasticsearch, Logstash, and Kibana) - something that we will use heavily in the future.

As many sipxcom users know, we at eZuce always use cutting edge technologies. In the current generation of software we benefit from a great configuration management tool called CFEngine, check these two links below: http://wiki.sipxcom.org/display/sipXcom/cfengine+tips and https://cfengine.com/company/blog-detail/qa-with-ezuces-michael-picher/

Let’s get hands on...

For this Proof of Concept (PoC) we used an Intel Core I5 with 8GB of RAM laptop that is running Fedora 22 with Ansible, Vagrant and VirtualBox installed. You can find plenty of information on the Internet about how to install these tools so there’s no need to add more here.

**Step 1. Create a Vagrantfile to bring up three sipXcom machines that will form the cluster.**

Since I encountered issues with USB 2.0 because of the VirtualBox Guest Additions used I had to disable that option with the following lines, where I’ve also allocated 2 GB of RAM per machine:


```
# disable USB 2.0 & add 2 GB/machine
         config.vm.provider "virtualbox" do |vb|
           vb.customize ["modifyvm", :id, "--usb", "off"]
           vb.customize ["modifyvm", :id, "--usbehci", "off"]
           vb.memory="2048"
         End
```




We also want to connect as root, in order to do that we need to add these instructions:

```
config.ssh.username = 'root'
config.ssh.password = 'vagrant'
config.ssh.insert_key = 'true'
```

Here comes the fun part where we will create 3 machines at once from a minimal centos 6 basebox:

```
# create cluster
       (1..3).each do |i|
    config.vm.define "uc#{i}" do |node|
        node.vm.box = "minimal/centos6"
        node.vm.hostname = "uc#{i}"
        node.vm.network :private_network, ip: "10.0.0.20#{i}"
```

Let’s explain what we are doing:

node.vm.box = "minimal/centos6"  --- We are using a minimal centos6 basebox as the OS needed for sipxcom

node.vm.hostname = "uc#{i}"    --- We are setting the names of the machines as uc1; uc2 and uc3

node.vm.network :private_network, ip: "10.0.0.20#{i}"   --- Assign 10.0.0.201-10.0.0.203 IP range for  those three servers we just created


Notes:
A. Due to a limitation of Vagrant/VirtualBox eth0 is needed by Vagrant to be used as NAT’ed interface (http://superuser.com/questions/752954/need-to-do-bridged-adapter-only-in-vagrant-no-nat)
B. Due a sipxcom limitation ” UC-4044 uniteme setup to bind network interface of choice”


We cannot use bridged interfaces so we will need to forward ports in order to achieve connections from outside NAT’ed LAN.


```
# Forwarding ports from guest due to eth0 limitation from Vagrant/Virtualbox
node.vm.usable_port_range = 2100..3250
node.vm.network "forwarded_port", guest: 21, host: "22#{i}1", id: 'ftp', auto_correct: true
node.vm.network "forwarded_port", guest: 22, host: "23#{i}2", id: 'ssh', auto_correct: true
node.vm.network "forwarded_port", guest: 80, host: "2#{i}80", id: 'web', auto_correct: true
node.vm.network "forwarded_port", guest: 443, host: "244#{i}", id: 'https', auto_correct: true
node.vm.network "forwarded_port", guest: 5060, host: "50#{i}60", protocol: 'udp', id: 'sip', auto_correct: true
node.vm.network "forwarded_port", guest: 5060, host: "50#{i}60", protocol: 'tcp', id: 'sip', auto_correct: true
```

**Step 2. Run Vagrant up to create those three machines**

You will get this output in your CLI


```
[mihai@localhost SIPXCOM_Cluster]$ vagrant up --provider=virtualbox
Bringing machine 'uc1' up with 'virtualbox' provider...
Bringing machine 'uc2' up with 'virtualbox' provider...
Bringing machine 'uc3' up with 'virtualbox' provider...
==> uc1: Importing base box 'minimal/centos6'...
==> uc1: Matching MAC address for NAT networking...
==> uc1: Checking if box 'minimal/centos6' is up to date...
==> uc1: Setting the name of the VM: SIPXCOM_Cluster_uc1_1472016831550_11979
==> uc1: Clearing any previously set network interfaces...
==> uc1: Preparing network interfaces based on configuration...
    uc1: Adapter 1: nat
    uc1: Adapter 2: hostonly
==> uc1: Forwarding ports...
    uc1: 21 (guest) => 2211 (host) (adapter 1)
    uc1: 22 (guest) => 2312 (host) (adapter 1)
    uc1: 80 (guest) => 2180 (host) (adapter 1)
    uc1: 443 (guest) => 2441 (host) (adapter 1)
    uc1: 5060 (guest) => 50160 (host) (adapter 1)
==> uc1: Running 'pre-boot' VM customizations...
==> uc1: Booting VM...
==> uc1: Waiting for machine to boot. This may take a few minutes...
    uc1: SSH address: 127.0.0.1:2312
    uc1: SSH username: root
    uc1: SSH auth method: password
    uc1: Warning: Remote connection disconnect. Retrying…
```


At the first run, downloading basebox  will take a few minutes but after that should be extremely fast

```
: ==> uc1: Importing base box 'minimal/centos6'...
```

After all vagrant tasks are finished  you can check machines status with vagrant status command:

```
[mihai@localhost SIPXCOM_Cluster]$ vagrant status

Current machine states:
uc1                       running (virtualbox)
uc2                       running (virtualbox)
uc3                       running (virtualbox)
```

This environment represents multiple VMs. The VMs are all listed above with their current state. For more information about a specific VM, run `vagrant status NAME`.

**Step 3. Prepare for Ansible by copying ssh keys with:**

```
[mihai@localhost SIPXCOM_Cluster]$ ssh-copy-id root@10.0.0.201
```


When prompted type “yes” and root password: 'vagrant' - defined in Vagrantfile
Perform same actions on all three nodes.

**Step 4. Let Ansible do its magic:**

Note : Ansible syntax  is extremely **sensible to indentation**. If your commands will not run from first make sure you have the correct indentation


We first need to define in the current path a file that will contain the hosts where we want to install sipxcom. This file is called inventory.ini and contains:

```
[uc1]
10.0.0.201
[uc2]
10.0.0.202
[uc3]
10.0.0.203
```

Next we will define ansible.cfg in the same current path instructing ansible to use root when connecting to the hosts defined in inventory.ini

```
[defaults]
hostfile = inventory.ini
remote_user = root
```

Final step we need to define a playbook that will install sipxcom.I’ve called this file install_sipxcom.yml

```
---
- hosts: all
  tasks:
    - name: Copy sipxcom.repo in /etc/yum.repos.d
      copy: src=sipxcom.repo dest=/etc/yum.repos.d/sipxcom.repo
    - name: Install openuc
      yum: name=sipxcom state=present
```

As you can see first we need to download sipxcom.repo file from http://download.sipxcom.org/pub/sipXecs/16.02/sipxecs-16.02.0-centos.repo . This file will be copied by ansible on /etc/yum.repos.d/ path on remote servers and after that will make sure it is installed

**Step 5. Run ansible playbook:**

Go get a coffee while sipxcom is installed, it will take a while …


After Ansible finishes the task you will see output that looks like:


```
[mihai@localhost SIPXCOM_Cluster]$ ansible-playbook install_sipxcom.yml


PLAY [all] *********************************************************************


TASK [setup] *******************************************************************
ok: [10.0.0.201]
ok: [10.0.0.202]
ok: [10.0.0.203]


TASK [Copy sipxcom.repo in /etc/yum.repos.d] ***********************************
ok: [10.0.0.201]
ok: [10.0.0.202]
ok: [10.0.0.203]


TASK [Install openuc] **********************************************************
changed: [10.0.0.202]
changed: [10.0.0.203]
changed: [10.0.0.201]


PLAY RECAP *********************************************************************
10.0.0.201                 : ok=3    changed=1    unreachable=0    failed=0
10.0.0.202                 : ok=3    changed=1    unreachable=0    failed=0
10.0.0.203                 : ok=3    changed=1    unreachable=0    failed=0
```

**Step 6. Run sipxecs-setup and connect to your newly installed cluster**

You will need to connect to your localhost IP on port 2441 ( guest: 443, host: "244#{i}", id: 'https',)

https://localhost:2441/sipxconfig/app

