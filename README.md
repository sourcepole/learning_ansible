create a virtual machine
------------------------

We set up a virtual machine that we're configuring via ansible.

* first [configure a local virtual machine](create_local_qemu_kvm_vm.md) to work with
  
* create yourself a clean new user and change into it

      $ sudo add user learnansible
      $ su - learnansible

clean up
--------

Remove the stuff we've created.

* deluser --remove-home learnansible
