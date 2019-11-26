create a virtual machine
------------------------

* make sure your machine supports virtualisation

  https://linuxwebdevelopment.com/run-debian-qemu-kvm-virtual-machine-using-ubuntu-debian/

* get required tools

      $ sudo apt-get install wget qemu-utils qemu-kvm libvirt-daemon-system

* make yourself a working directory

      $ mkdir vm-handling
      $ cd    vm-handling

* get a Debian install medium (335M):
  
      $ wget https://cdimage.debian.org/cdimage/release/current/amd64/iso-cd/debian-10.2.0-amd64-netinst.iso

* create a "hard-disk" image
  
      $ qemu-img create debian.img 2G

### create a virtual machine and boot

* start a virtual machine with that ISO file

      $ kvm -hda debian.img -cdrom debian-10.2.0-amd64-netinst.iso -boot d -m 512 -net user -net nic

* go through install

  * return, return, return ...
  * set root password, add a random user
  * return, return, return
  * choose "yes" to write disk partitioning
  * return, return, return
  * don't select any additional software to install, tab out to "OK"
  * return
  * choose sda to install booloader on
  * reboot, stop the kvm machine

* restart installed virtual machine without cdrom, but with networking
  localhost:5555 will get forwarded to the vm-internal port 22

      $ kvm -hda debian.img -m 512 -net user -device e1000,netdev=net0 -netdev user,id=net0,hostfwd=tcp::5555-:22

* log in as root in qemu window

      # vi /etc/ssh/sshd_config
      PermitRootLogin yes
      # /etc/init.d/ssh restart
  
* now you can ssh into the machine from your local shell

      # ssh localhost -p 5555

