# Ovs 2.15.4 with kernel version 5.15.0.86

## 1. Bring up ubuntu vm with 22.04 with attached vagrant file

## 2. Install below dependencies/ pre-requisites
    sudo apt update
    sudo apt install -y build-essential linux-headers-generic
    sudo reboot    ---> to update to the latest kernel version 5.15.0.86 (if required)
    sudo apt install -y dh-make debhelper dh-python devscripts python3-dev
    sudo apt install -y graphviz libssl-dev python3-all python3-sphinx libunbound-dev libunwind-dev
    sudo apt-get install libelf-dev
    sudo apt-get install module-assistant
    sudo apt install net-tools
    sudo apt-get install graphviz autoconf automake bzip2 debhelper dh-autoreconf libssl-dev libtool openssl procps python3-all python3-sphinx python3-twisted python3-zope.interface libunbound-dev libunwind-dev dh-python -y
    sudo apt-get update
    sudo apt-get install debhelper
    sudo apt  install ruby-rubygems
    sudo gem install fpm
    sudo apt-get install dctrl-tools dkms libcharon-extauth-plugins libelf-dev libstrongswan libstrongswan-standard-plugins strongswan strongswan-charon strongswan-libcharon strongswan-starter zlib1g-dev
    sudo apt-get -y install "linux-headers-$(uname -r)"
## 3. run the script pre_build.sh to apply ovs patch for magma agw
    
     mkdir -p /home/vagrant/ovs_deb_pkg                 >  ---> folder, where debian package files will be created.
     sudo bash -x pre_build.sh /home/vagrant/ovs_deb_pkg
## 4. ovs 2.15.4 patch for kernel 5.15.x
     apply attached patch file fix-ovs-kernel-5.15.x_diff using below command
         patch -p1 < fix-ovs-kernel-5.15.x_diff                             --------> run it from the ovs repo folder.
## 5. create the ovs debian packages using post_build.sh.
    
     sudo bash -x post_build.sh /home/vagrant/ovs_deb_pkg
     remove ovs ipsec debian package file as ovs restart will fail due to this
     rm /home/vagrant/ovs_deb_pkg/openvswitch-ipsec_2.15.4-10-magma_amd64.deb
     sudo dpkg -i *.deb            ----> apply ovs debian package
     sudo dpkg --configure -a
     sync
     depmod -a

## 6. To verify ovs is installed with gtp modules and in running state before agw installation.
     sudo apt list --installed | grep openvs
     sudo ovs-vsctl show
     lsmod | grep gtp
     below is the expected output:
          vport_gtp              16384  1
          openvswitch           208896  9 vport_gtp
          gtp                    28672  0
          udp_tunnel             20480  3 openvswitch,gtp,sctp
     if gtp and vport_gtp modules are not loaded, please run below commands to load the modules
         sudo insmod /lib/modules/5.15.0-86-generic/updates/dkms/vport-gtp.ko
         sudo apt-get install linux-modules-extra-$(uname -r)
         sudo modprobe gtp
     check ovs status
         sudo systemctl status openvswitch-switch
      restart ovs before agw installation
         sudo systemctl stop openvswitch-switch
         sudo systemctl start openvswitch-switch
     
## 7. steps to install dockerized agw
        mkdir -p /var/opt/magma/certs
        sudo vim /var/opt/magma/certs/rootCA.pem                ----> Add your rootCA.pem obtained from orc8r
        openssl x509 -text -noout -in /var/opt/magma/certs/rootCA.pem
        Fetch the Docker Scripts
          wget https://github.com/magma/magma/raw/master/lte/gateway/deploy/agw_install_docker.sh
        Rename the interfaces to eth0,eth1 and eth2
           sudo su
           sed -i 's/GRUB_CMDLINE_LINUX=""/GRUB_CMDLINE_LINUX="net.ifnames=0 biosdevname=0"/g' /etc/default/grub
           sed -i 's/enp0s3/eth0/g' /etc/netplan/50-cloud-init.yaml
           grub-mkconfig -o /boot/grub/grub.cfg
           exit
           exit
           vagrant reload distromagma
           vagrant ssh distromagma
           sudo chmod 777 agw_install_docker.sh
           sudo bash agw_install_docker.sh
     Launch the dockers
           cd /var/opt/magma/docker/
           sudo ./agw_upgrade.sh
     once agw is up, verify the docker containers, sctp port and gtp modules for ovs
           lsmod | grep gtp
           sudo docker ps -a
           cat /proc/net/sctp/eps
   Establish the PDU session and test traffic.
       
