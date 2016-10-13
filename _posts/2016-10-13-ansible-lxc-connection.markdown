---
layout: post
title:  "Orchestrating SSH-less LXC containers using Ansible"
date:   2016-10-13 10:32:00 -0500
categories: ansible devops deployment automation lxc containers
---

## The catch with LXC containers and Ansible

When you execute an Ansible playbook, modules you use in the tasks you define are compiled to a Python script, copied to the remote system via SSH, and then executed via SSH.

Ansible does this using its built in ssh connection plugin, which does the dirty work of copying and executing the files, running commands in sudo, etc. When you're dealing with bare-metal hosts or VMs, the hosts you're managing probably already have SSH running, so that's a great way to manage a wide variety of infrastructure without having to add yet another agent.

However, when managing LXC containers, perhaps you do not wish to have SSH running inside each and every container, connect each container to your management network, and deal with the security implications of managing all of these SSH installations.

## Meet the lxc_ssh connection plugin

This guide uses the [lxc_ssh](https://github.com/chifflier/ansible-lxc-ssh) as an example, however another ssh plugin is availble in the [Openstack-Ansible plugins repo](https://github.com/openstack/openstack-ansible-plugins/blob/master/connection/ssh.py).

Either one of these plugins will allow us to execute tasks in Ansible targeting the container without accessing the container over SSH. Instead, the host that the container lives on will receive the SSH connection, and the Ansible module will be transparently executed using lxc-attach.

## How do I set up lxc_ssh?

Take for example a host named "vm" which has a container named "container" running on it. The container does not have 'sshd' running, however I do have root SSH access to "vm" from the remote host I am executing my Ansible playbooks on and I just wrote some plays I want to run on "container".

First, I need to download the lxc_ssh connection plugin:
{% highlight bash %}
mkdir connection_plugins
wget -O connection_plugins/lxc_ssh.py https://raw.githubusercontent.com/chifflier/ansible-lxc-ssh/master/lxc_ssh.py
{% endhighlight %}

And now my project looks like this:
{% highlight plaintext %}
ansible-project/
├── connection_plugins
│   └── lxc_ssh.py
└── playbook.yml
{% endhighlight %}

Alongside playbook.yml, create a file called ansible.cfg and configure it with the path to our lxc_ssh connection plugin and test inventory.
{% highlight bash %}
echo -e '[defaults]\nconnection_plugins = connection_plugins\ninventory = inventory' > ansible.cfg
{% endhighlight %}

Create your inventory (replace the ansible_host setting with the IP/hostname of "vm"):
{% highlight bash %}
mkdir -p inventory/group_vars
echo -e '[hosts]\nvm ansible_host=10.10.155.50' >> inventory/all
echo -e '[containers]\ncontainer physical_host=vm' >> inventory/all
{% endhighlight %}

Now create `inventory/group_vars/containers.yml`. This maps "container" to "vm" and enables the `lxc_ssh` connection plugin for the container host entries:
{% highlight yaml %}
---
ansible_host: "{%raw%}{{ physical_hostname }}{%endraw%}"
ansible_connection: lxc_ssh
ansible_ssh_extra_args: "{%raw%}{{ container_name }}{%endraw%}"
ansible_user: root
container_name: "{%raw%}{{ inventory_hostname }}{%endraw%}"
physical_hostname: "{%raw%}{{ hostvars[physical_host]['ansible_host'] }}{%endraw%}"
{% endhighlight %}

And last, an example playbook for `playbook.yml`:
{% highlight yaml %}
- name: Test Playbook
  hosts: containers
  gather_facts: true
  tasks:
    - name: testfile
      copy:
        content: |
          testing file
        dest: /tmp/test
    - name: test echo
      command: cat /tmp/test
      register: echo
    - name: test hostname
      command: hostname
      register: hostname
    - name: print file contents
      debug:
        msg: "{%raw%}{{ echo.stdout }}{%endraw%}"
    - name: print remote hostname
      debug:
        msg: "{%raw%}{{ hostname.stdout }}{%endraw%}"
{% endhighlight %}

This playbook will run inside the container, as confirmed by the output of `hostname`.

So the final layout of our test project will look like:
{% highlight plaintext %}
ansible-project/
├── ansible.cfg
├── connection_plugins
│   └── lxc_ssh.py
├── inventory
│   ├── all
│   └── group_vars
│       └── containers.yml
└── playbook.yml
{% endhighlight %}

And.. our test output!
{% highlight plaintext %}
$ ansible-playbook playbook.yml

PLAY [Test Playbook] ***********************************************************

TASK [setup] *******************************************************************
ok: [container]

TASK [testfile] ****************************************************************
ok: [container]

TASK [test echo] ***************************************************************
changed: [container]

TASK [test hostname] ***********************************************************
changed: [container]

TASK [print file contents] *****************************************************
ok: [container] => {
    "msg": "testing file"
}

TASK [print remote hostname] ***************************************************
ok: [container] => {
    "msg": "container"
}

PLAY RECAP *********************************************************************
container                  : ok=6    changed=2    unreachable=0    failed=0
{% endhighlight %}
