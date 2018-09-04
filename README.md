# Vagrant Setup for tendrl

This setup is a modified version of Tendrl-Vagrant Upstream( 
https://github.com/Tendrl/tendrl-vagrant by 
[Shirshendu Mukherjee](https://github.com/shirshendu) ) .

This will setup as many gluster storage nodes as you want with a number of bricks that you can define!
It works with your local tendrl-api server and tendrl-ui setup.
It uses vagrant's ansible plugin, so you will need ansible (>=2.4) on your host machine.

## Requirements
* macOS with [Virtualbox](https://www.virtualbox.org/wiki/Downloads) (starting 5.1.30) **or**
* RHEL 7.4/Fedora 27 with KVM/libvirt
* [Ansible](https://ansible.com) (starting 2.4.0.0)
* [Vagrant](https://www.vagrantup.com) (starting 1.9.1)
* git

## Installation instructions for Vagrant / Ansible

#### On RHEL 7.4

* make sure you are logged in as a user with `sudo` privileges
* make sure your system has the following repositories enabled (`yum repolist`)
  * rhel-7-server-rpms
  * rhel-7-server-extras-rpms
* install the requirements
  * `sudo yum groupinstall "Virtualization Host"`
  * `sudo yum install ansible git gcc libvirt-devel`
  * `sudo yum install https://releases.hashicorp.com/vagrant/2.0.1/vagrant_2.0.1_x86_64.rpm`
* start `libvirtd`
  * `sudo systemctl enable libvirtd`
  * `sudo systemctl start libvirtd`
* enable libvirt access for your current user
  * `sudo gpasswd -a $USER libvirt`
* as your normal user, install the libvirt plugin for vagrant
  * `vagrant plugin install vagrant-libvirt`

#### On Fedora 27

* make sure you are logged in as a user with `sudo` privileges
* make sure your system has the following repositories enabled (`dnf repolist`)
  * fedora
  * fedora-updates
* install the requirements
  * `sudo dnf install ansible git gcc libvirt-devel libvirt qemu-kvm`
  * `sudo dnf install vagrant vagrant-libvirt`
* start `libvirtd`
  * `sudo systemctl enable libvirtd`
  * `sudo systemctl start libvirtd`
* enable libvirt access for your current user
  * `sudo gpasswd -a $USER libvirt`
* as your normal user, install the libvirt plugin for vagrant
  * `vagrant plugin install vagrant-libvirt`

#### On macOS High Sierra

* install the requirements
  * install [Virtualbox](https://www.virtualbox.org/wiki/Downloads)
  * install [Vagrant](https://www.vagrantup.com)
  * install [homebrew](https://brew.sh/)
  * install git
    * `brew install git`
  * install ansible
    * `brew install ansible`
    
## Features 
* Can be deployed on local system or on a remote machine.
* Can be shared with others.

## Note
* Please go through the complete document below before starting the deployment.
* The VM’s created by this setup are of Centos base image (RHEL will be updated soon)
* It would be better to run commands as root user
* It is recommended to make a Private Setup for local system
* Public setup is necessary if the host machine is a remote machine (eg lab machines)
* Private setup cannot be shared with others but public can be.



## Get started
* Clone this repository
  * `git clone https://github.com/anmolsachan/tendrl-vagrant/tree/downstream`
* Goto the folder in which you cloned this repo
  * `cd tendrl-vagrant`
* if you are a returning user run `git pull` to ensure you have the latest updates
* if you are on RHEL/Fedora and you don't want your libvirt storage domain `default` to be used, override the storage domain like this
  * `export LIBVIRT_STORAGE_POOL=images`
* `cp tendrl.conf.yml.sample tendrl.conf.yml`
  * Decide how many storage nodes and how many bricks you need
  * Decide if you want vagrant to initialize the cluster (`gdeploy`) for you
  * If you opted to initialize the cluster, decide whether you want to deploy tendrl
  * edit options in this file
* Configure tendrl repos :
    * open file ./ansible/prepare-gluster.yml
    * For all the required repos provide `url` and `dest` fields accordingly in the file.
* Configuring vagrant to bridge with the host’s network ( Important )
    * Creating a PRIVATE Setup (Default)
        * No changes required.
    * Creating a PUBLIC Setup
        * This setup creates public VM’s which acquire IP’s from DHCP to which the host machine is connected.
        * (Requirement/ limitation)This requires your host machine to be connected to be connected to a wired connection (Ipv4) like eth0 , em1, etc.
        * Replace `config.vm.network 'private_network', type: 'dhcp', auto_config: true` On line 140 of Vagrantfile with
            ```
                config.vm.network "public_network", :bridge => 'em1', :dev => 'em1'
                config.vm.network "forwarded_port", guest: 80, host: 8080
                config.vm.network "forwarded_port", guest: 2379, host: 2379
            ```
        * Configure the parameters `:bridge`  and `:dev`
            * `config.vm.network "public_network", :bridge => 'em1', :dev => 'em1'`
        * To find the network interface which your host machine has do `ip a` or `ip link show`
        * Copy the interface which provides public (lan) ip to your device eg: eth0, em0, em1, etc.
        * Uncomment the below lines (50-60) in file ./ansible/prepare-environment.yml
            ``` 
            - name: Set authorized key taken from know hosts for vagrant
              authorized_key:
                user: vagrant
                state: present
                key: "{{ lookup('file', '/root/known_machines.pub') }}"

            - name: Set authorized key taken from know hosts for root
              authorized_key:
                user: root
                state: present
                key: "{{ lookup('file', '/root/known_machines.pub') }}"
            ```
* VM spec configurations (#optional):
    * VM spec configurations (#optional):
        * tendrl storage nodes
            * Number of disks configured in tendrl.conf.yml
        * tendrl server node :
            * 3 disks :
                * OS
                * Mount point for etcd
                * Mount point for carbon
            * You can configure the size of the etcd and carbon disks (but not OS) in Vagrantfile. 
* Provision to share the setup (For PUBLIC setup only):
    * Create file `/root/known_machines.pub` (default path)on the host machine.
    * If you do not want to use the path `/root/known_machines.pub`    
    Configure the path at location in the downloaded folder : /ansible/prepare-environment.yml  ---> lines 54 and line 60.
    * Copy your the public key ( ~/.ssh/id_rsa.pub ) into the file root/known_machines.pub . You can add public keys of other users to the file.
    * If you want to share your setup with users after the setup is created, you will have to add the public keys of other users at /root/.ssh/authorized_keys  in all the nodes.

* Run `vagrant up`
  * Wait a while

## Usage
* *Always make sure you are in the git repo - vagrant only works in there!*
* After `vagrant up` you can connect to each VM with `vagrant ssh` and the name of the VM you want to connect to
* The single central server VM is called `tendrl-server`
* Each storage node VM is called `tendrl-node-x` where x starts with 1
  - tendrl-node-1 is your first VM and it counts up depending on the amount of VMs you spawn
* If you get the following error:

`Error while activating network: Call to virNetworkCreate failed: error creating bridge interface virbr0: File exists.`

Please try restarting `libvirtd` with `sudo systemctl restart libvirtd`

* There are also other vagrant commands you should check out!
  * if you want to throw away everything: `vagrant destroy -f`
  * if you want to freeze the VMs and continue later: `vagrant suspend`
  * Try `vagrant -h` to find out about them
  * if you run `vagrant up` again you without running `vagrant destroy` before you will overwrite your configuration and vagrant may loose track of some VMs (it's safe to remove them manually)
* modify the `VMMEM` and `VMCPU` variables in the Vagrant file to change the VM resources, adjust `VMDISK` to change brick device sizes

## For more info visit: 
* https://github.com/Tendrl/tendrl-vagrant
* https://www.vagrantup.com/docs/

## ADD-ON (Optional) - Heketi deployment for creating and managing volumes

* By default this setup creates volumes using gdeploy.
* If you want to create volumes using heketi , you need to make changes to the following files to remove volume creation by gdeploy ( Needs to be done before running vagrant up)
    * In ansible/templates
        * File gdeploy-libvirt.conf.j2  and file gdeploy-virtualbox.conf.j2, remove following
            ```
            [backend-setup]
            devices={{ devices|join(",") }}
            brick_dirs=brick{1..{{ devices|length }}}
            
            [volume]
            action=create
            replica=yes
            replica_count=2
            force=yes
            ```
* Steps to deploy Heketi on the tendrlc server node :
    * Ssh to server node
    * DO 
        ``` 
        $ yum install wget -y

        $ wget https://github.com/heketi/heketi/releases/download/v6.0.0/heketi-v6.0.0.linux.amd64.tar.gz
        
        $ tar xzvf heketi-v6.0.0.linux.amd64.tar.gz
        
        $ mkdir /var/lib/heketi

        ```
    * Configure heketi.json
        * Change directory to heketi folder
        * There is already a config file present in the setup directory (heketi/heketi.json). You can replace the heketi.json in pwd with that.
        `export HEKETI_CLI_SERVER=http://localhost:8080`
    * Run heketi server
        `./heketi --config=heketi.json`
    * In a new terminal , go to same extracted heketi directory
        * Create a file named topology_libvirt.json
        * You can replace the above with the file present in the setup directory (heketi/topology_libvirt.json).
        * For each node provide storage node ips and device info in the topology_libvirt.json
        * Load the topology by :
        `./heketi-cli topology load --json=topology_libvirt.json`
        
    * Create volume eg : `./heketi-cli volume create --size=1 --replica=2`
        

#### Cheers !

## Author
[Anmol Sachan](https://github.com/anmolsachan)

## Orignal Author
[Shirshendu Mukherjee](https://github.com/shirshendu)

## Ultimate Orignal Authors
[Christopher Blum](https://github.com/zeichenanonym)

[Daniel Messer](mailto:dmesser@redhat.com) - [dmesser@redhat.com](mailto:dmesser@redhat.com) -
Technical Marketing Manager @ Red Hat
