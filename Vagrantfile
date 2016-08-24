
Vagrant.configure(2) do |config|
config.ssh.username = 'root'
config.ssh.password = 'vagrant'
config.ssh.insert_key = 'true'
 
      # disbale USB 2.0 add 2 GB/machine
      config.vm.provider "virtualbox" do |vb|
           vb.customize ["modifyvm", :id, "--usb", "off"]
           vb.customize ["modifyvm", :id, "--usbehci", "off"]
           vb.memory="2048"
      end


      
       # create cluster 
 (1..3).each do |i|
    config.vm.define "uc#{i}" do |node|
        node.vm.box = "minimal/centos6"
        node.vm.hostname = "uc#{i}"
        node.vm.network :private_network, ip: "10.0.0.20#{i}"
        
        # Forwarding ports from guest due to eth0 limitation from Vagrant/Virtualbox
         node.vm.usable_port_range = 2100..3250
         node.vm.network "forwarded_port", guest: 21, host: "22#{i}1", id: 'ftp', auto_correct: true
         node.vm.network "forwarded_port", guest: 22, host: "23#{i}2", id: 'ssh', auto_correct: true
         node.vm.network "forwarded_port", guest: 80, host: "2#{i}80", id: 'web', auto_correct: true
         node.vm.network "forwarded_port", guest: 443, host: "244#{i}", id: 'https', auto_correct: true
         node.vm.network "forwarded_port", guest: 5060, host: "50#{i}60", protocol: 'udp', id: 'sip', auto_correct: true
         node.vm.network "forwarded_port", guest: 5060, host: "50#{i}60", protocol: 'tcp', id: 'sip', auto_correct: true           
    end    
 end



end
