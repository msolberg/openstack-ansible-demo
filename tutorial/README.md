= Using Ansible to Launch Instances on OpenStack 

This is a short tutorial on how to deploy instances onto OpenStack
clouds using Ansible.  It assumes familiarity with the [Ansible
Playbook
Introduction](http://docs.ansible.com/ansible/playbooks_intro.html).
Start there if you haven't written an Ansible playbook before.  The
goal of this tutorial is to launch and configure an instance on an
OpenStack platform.  It assumes that the instance does not exist in
the current inventory of the Ansible host.

== Start simple.

We'll begin with a simple playbook that has a single play - a
simplified version of the webserver example in the Ansible Playbook
Introduction.  This play follows.

```
- hosts: webservers
  remote_user: centos
  sudo: yes
  tasks:
    - name: ensure apache is at the latest version
      yum: name=httpd state=latest
    - name: make sure apache is running
      service: name=httpd state=running
```

We're running two tasks in this play - one to install apache and one
to start it and enable it.  We're also specifying that this play
should be run against any hosts in the webservers group and should be
run as the `centos` user.

To run the first play, manually launch a CentOS image in your
OpenStack tenant and assign a floating IP to it.  Then create a simple
hosts file with the floating IP listed in the "webservers" group:

```
[webservers]
<floating_ip>
```

Run the example playbook like so:

```
$ ansible-playbook -i hosts webserver.yml
```

You should see that the tasks succeed and that the Apache HTTP Server
is installed and running on the instance.  If they don't, check that
you've got your SSH keys configured correctly, the instance can access
the Internet to download packages, you have connectivity to the
instance, and so on.

== Launching an Instance

Next we'll launch an instance in an Ansible task.  We'll be using the
`nova_compute` module available in Ansible 1.9.  Starting with Ansible
2.0, this module has been deprecated in favor of the much improved
os_server module.

The following play demonstrates basic usage of the nova_compute module.

```
- name: Deploy on OpenStack
  hosts: localhost
  gather_facts: false
  tasks:
  - name: Deploy an instance
    nova_compute:
      state: present
      login_username: demo
      login_password: secret
      login_tenant_name: demo
      auth_url: http://127.0.0.1:5000/v2.0/
      name: webserver
      image_name: centos-7-x86_64-genericcloud
      key_name: root
      wait_for: 200
      flavor_id: 2
      auto_floating_ip: yes
      nics:
        - net-id: fa6af4e6-c44e-439c-a91c-03bcae55e587
      meta:
        hostname: webserver.localdomain
```

There are a couple of things to note with this example.  The first is
that every task in Ansible needs to run on some host.  The `nova_compute`
task is typically a "local action", but doesn't have to be.  Whichever
machine you're going to run the task on needs to have the Python Nova
Client library installed (`python-novaclient` on Red Hat
distributions).  Second, we're turning off fact gathering.  This also
isn't necessary, but speeds up the execution quite a bit.  Also
consider specifying that your connection to localhost is local with a
line like the following in your ansible hosts file:

```
localhost        ansible_connection=local
```

The task itself is pretty simple.  It contains all of the variables
that you would expect to specify to launch an instance.  Edit them to
match your environment and run the playbook with the following command.

```
ansible-playbook openstack_deploy.yaml
```

Assuming that you've specified the correct network ID, image name, key
name, and so on, you should see the following output and have an
instance named "webserver" running in your tenant now.

```
PLAY [Deploy on OpenStack] **************************************************** 

TASK: [Deploy an instance] **************************************************** 
changed: [localhost]

PLAY RECAP ******************************************************************** 
localhost                  : ok=1    changed=1    unreachable=0    failed=0   

```

Note that the `nova_compute` module is idempotent.  Running this
playbook multiple times will not launch multiple instances.  Also note
that the instance can be terminated by changing the "state" attribute
from "present" to "absent" and running the playbook.

== Putting it all together.

Now that we can launch an instance in OpenStack, the next logical step
is to configure it.  First, we'll use the `add_host` module to add the
provisioned instance into the Ansible inventory after we provision it.
This is a little complicated, but we can't really know what the
floating IP of the instance will be until we've provisioned it, so we
can't add it to the inventory before we run the playbook.  The example
follows.

```
- name: Deploy on OpenStack
  hosts: localhost
  gather_facts: false
  tasks:
  - name: Deploy an instance
    nova_compute:
      state: present
      login_username: demo
      login_password: secret
      login_tenant_name: demo
      auth_url: http://127.0.0.1:5000/v2.0/
      name: webserver
      image_name: centos-7-x86_64-genericcloud
      key_name: root
      wait_for: 200
      flavor_id: 2
      auto_floating_ip: yes
      nics:
        - net-id: fa6af4e6-c44e-439c-a91c-03bcae55e587
      meta:
        hostname: webserver.localdomain
    register: nova

  - name: Add CentOS Instance to Inventory
    add_host: name=webserver groups=webservers
              ansible_ssh_host={{ nova.public_ip[0] }} 
  
- hosts: webservers
  remote_user: centos
  sudo: yes
  tasks:
    - name: ensure apache is at the latest version
      yum: name=httpd state=latest
    - name: make sure apache is running
      service: name=httpd state=running
```

Once again, modify the attributes in the `nova_compute` task to match
your environment.  These task is the same as the one we used in the
last section, except that we're now registering the output of the task
so that we can access the floating IP address of the instance as a
variable.  The second task in the "Deploy on OpenStack" play is where
we dynamically add the host to the inventory.

The last play in the playbook is the same the play that we started the
tutorial with.  It installs and starts apache on all of the
webservers, which now includes the instance that we provisioned on
OpenStack.  Run this example the same way as the last one:

```
$ ansible-playbook openstack_webserver.yaml
```

You should get some output similar to the following:

```
PLAY [Deploy on OpenStack] **************************************************** 

TASK: [Deploy an instance] **************************************************** 
ok: [localhost]

TASK: [Add CentOS Instance to Inventory] ************************************** 
ok: [localhost]

PLAY [webservers] ************************************************************* 

GATHERING FACTS *************************************************************** 
ok: [webserver]

TASK: [ensure apache is at the latest version] ******************************** 
changed: [webserver]

TASK: [make sure apache is running] ******************************************* 
changed: [webserver]

PLAY RECAP ******************************************************************** 
localhost                  : ok=2    changed=0    unreachable=0    failed=0   
webserver                  : ok=3    changed=2    unreachable=0    failed=0   
```

Note that there are now two hosts which are listed in the Play recap -
the plays which launch the instance are run on localhost and the plays
which configure the webserver are run on the provisioned instance.

