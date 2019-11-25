Learning ansible
================

This is a intermediate level to learn ansible from scratch.

The course is made to work on Debian based systems (Ubuntu, Mint...)
but should be easily transferable to other Linux distributions or
Unices.

create a virtual machine
------------------------

We set up a virtual machine that we're configuring via ansible.

* first [configure a local virtual machine](create_local_qemu_kvm_vm.md) to work with
  
* create yourself a clean new user and change into it

      $ sudo add user learnansible
      $ su - learnansible

copy your ssh key over to the VM
--------------------------------

* create a ssh key pair

      learnansible:~$ ssh-keygen

* copy your new ssh public key over to the vm

      learnansible:~$ ssh-copy-id -p 5555 root@localhost

* test that connecting to the VM works without a password prompt

      learnansible:~$ ssh root@localhost -p 555

* configure ssh for easy ssh access to the vm

      learnansible:~$ vim ssh root@localhost -p 555
      Host vm
      HostName localhost
      Port 5555
      User root

* test that you can now log into the vm quickly

      learnansible:~$ ssh vm

preparations...
---------------

* make yourself a working directory for ansible

      learnansible:~$ mkdir ansible
      learnansible:~$ cd ansible

* from here on, we'll allways be working as the `learnansible` user
  inside the `~/ansible` directory

      learnansible:~/ansible$ pwd
      /home/learnansible/ansible

* install ansible

      $ sudo apt-get install ansible

define the ansible inventory
----------------------------

* create the inventory of the machines ansible will take care of

      $ mkdir inventory
      $ vim inventory/production
      vm

* tell ansible about where it's resources are

      $ vim ansible.cfg
      [defaults]
      inventory = /home/learnansible/ansible/inventory/production

ansible in ad-hoc mode
----------------------

* your hello world ansible

      $ ansible vm -m ping
      vm | SUCCESS => {
          "changed": false,
          "ping": "pong"
      }

What happened here? We have called ansible in the ad-hoc mode.
We have told ansible where to execute a command - namely on
the `vm` machine. And we have told what to do on that machine
`-m ping`, which means "execute the module `ping`".

What the ping module does, is that it gets executed on the
*remote* machine, and emits a "pong"?

The result is green, so everything is OK, also ansible
explicitly tells us "SUCCESS".

We also see that the ping module didn't "changed: false" anything
on the remote machine (it only emitted a "pong").

Instead of telling ansible to execute the module on the
`vm` machine, we could tell ansible to execute it on
"all machines it knows about":

    $ ansible all -m ping

Try that out.

How does ansible know how to connect to the `vm` machine?
It uses `ssh` by default to connect to a machine, and we
have configured `ssh` previously to know about `Host vm`.

There are other modules...

the command module
------------------

The command module is the default ansible module. So
we can write:

    $ ansible vm -m command -a "echo bla"

(Execute that!)

Or, because `command` is the default module, we can
omit the `-m command` parameter:

    $ ansible vm -a "echo bla"

What's `-a "echo bla"`? It's the parameter we give to
the module. The `command` module expects to be given
the command to execute, which is `echo` followed by
whatever we would like to pass on to the `echo` command.

other modules
-------------

There are [*many* more modules](https://docs.ansible.com/ansible/latest/modules/modules_by_category.html).

executing on many machines
--------------------------

Until now, it seems like ansible is just a pathetic wrapper
around ssh. But we're just starting. We can define groups
of machines (and groups of groups) for ansible to execute
stuff on. So we can...

    $ vim inventory/production
    [webservers]
    vm
    othermachine

... define a group and tell ansible to execute on that
group:

    $ ansible webservers -a 'echo bla'
    vm | CHANGED | rc=0 >>
    bla
    
    othermachine | CHANGED | rc=0 >>
    bla

Unfortunately we do not have a `othermachine` but we
can fake it, by defining it exactly as our first `vm`
machine:

    $ vim ~/.ssh/config
    [...]
    Host othermachine
    HostName localhost
    Port 5555
    User root

That connects to our original VM, but ansible doesn't
know or care, so...

(Try `ansible webservers -a 'echo bla'` now)

Ansible has syntax to express on which machines and
groups you want to have stuff executed (or not).

inventories
-----------

The inventory above is just one form of inventory that
ansible can use to find out about its managed machines.
Ansible has [inventory plugins](https://docs.ansible.com/ansible/latest/plugins/inventory.html?highlight=inventory%20plugins)
that let you access read diverse inventory formats,
also dynamic ones, which allow you to access inventories
in live databases or other configuration management systems.

configuring machines
--------------------

If we want to configure more that one thing on a
(set of) machine(s) then we want to use playbooks.

A simple playbook that checks whether another
machine is reachable via the same `ping` module
as before looks like this:

    $ vim available.yml
    - hosts: vm
      tasks:
        - name: is the machine there?
          ping:

You'll notice the correspondence with the command:

    $ ansible vm -m ping

Also, you'll maybe notice, that the format of the playbook
is [YAML](https://yaml.org/).

You can run that playbook:

    $ ansible-playbook available.yml 
    
    PLAY [vm] ********************************************
    
    TASK [Gathering Facts] *******************************
    ok: [vm]
    
    TASK [is the machine there?] *************************
    ok: [vm]
    
    PLAY RECAP *******************************************
    vm         : ok=2    changed=0    unreachable=0    failed=0

Ansible executes the play, executes a "Gathering Facts" task
and then our "is the machine there?" task.

In the end it shows us a summary of the results of the
actions taken.

information gathering
---------------------

What's that "Gathering Facts" task we've just seen? It's
ansible finding out things about the target system.

We can run this fact gathering step standalone...

    $ ansible vm -m setup
    vm | SUCCESS => {
        "ansible_facts": {
            "ansible_all_ipv4_addresses": [
                "10.0.2.15"
            ],
            "ansible_all_ipv6_addresses": [
                "fec0::5054:ff:fe12:3456",
    [...]

... and it outputs heaps of infos about the system,
which could be reused to f.ex. feed some other database.

Within the context of an ansible play it can be used
to make decisions:

conditionals, variables
--------------------

Ansible's `gather_facts` aka `setup` collects
information that we can reuse in plays:

    $ vim is_buster.yml
    - hosts: vm
      tasks:
        - name: tell user that we're on buster
          debug: msg="whoa, we're on Debian buster"
          when: ansible_distribution_release == 'buster'

    $ ansible-playbook is_buster.yml
    PLAY [vm] ********************************************
    
    TASK [Gathering Facts] *******************************
    ok: [vm]
    
    TASK [tell user that we're on buster] ****************
    ok: [vm] => {
        "msg": "whoa we're on Debian buster"
    }
    
    PLAY RECAP *******************************************
    vm         : ok=2    changed=0    unreachable=0    failed=0   

We are using the `debug` module here, that accepts a `msg`
parameter, which it outputs.

Also the task is executed conditionally, depending on whether
the `when:` condition is true or false.

In this case we're checking whether ansible's
`ansible_distribution_release` variable is equal to the
value `buster`.

The `ansible_distribution_release` variable gets set by
ansible's `gather_facts` aka `setup` task.

Jinja2
------

What follows the `when:` ansible keyword is a "jinja2 expression".
[Jinja2](https://jinja.palletsprojects.com/) is a Python
templating language. That means that all that jinja2 can
do, we can use for conditionals.

But we can use jinja2 expressions more or less anywhere in
a playbook (or in a task or in a role, more later):

    - hosts: vm
      tasks:
        - name: tell user that we're on buster
          debug: msg="whoa, we're on Debian {{ ansible_distribution_release }}"

This will do what you think it will (try it out!). If
ansible finds a double quoted string with a "{{ }}" element
then everything inside the curly brackets will be interpreted
as a jinja expression. So we might write...

    [...]
    debug: msg="Are we on buster - {{ ansible_distribution_release == 'buster' }}"

...and we'll get:

    [...]
    ok: [vm] => {
        "msg": "Are we on buster - True"
    }

copy module
-----------

Let's have a look at a different module from the mentioned
available modules, f.ex.
[copy](https://docs.ansible.com/ansible/latest/modules/copy_module.html):

    $ vim copy_secret.yml
    - hosts: vm
      tasks:
        - name: copy database access password
          copy: src=secret dest=secret



clean up
--------

Remove the stuff we've created.

* deluser --remove-home learnansible
