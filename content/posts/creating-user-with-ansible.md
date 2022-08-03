---
title: "Creating user with Ansible"
date: 2022-08-03T21:31:38+02:00
---

I usually write code. Sometimes I run tools in the background. From time to time I process data that requires lots of resources.   
To do all of those things I need to have some specific environment. While some of this stuff can be done on my personal machine, there are cases when one needs to set up a remote machine.  
And since this is mostly for personal needs I don't want to leverage any complicated tools and stay minimalistic.  

The goal of this piece is to use Ansbile to create a user account on a remote machine with sudo capabilities and set up a secure connection to it, with no passwords, just SSH keys generated during the process.

Please, keep in mind this is an opinionated approach for my use case: I use [1password to generate and store root keys](https://developer.1password.com/docs/ssh/), and I upload the public key of the root to the VPS provider (I'm going to use Ubuntu on Vultr) so that I can deploy a server with root key already there.

## Bootstrapping VPS connection

Assuming the VPS is up (thus the root key is uploaded too) one can create an [inventory file](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html).  
Before doing so I'm going to create an alias in ~/.ssh/config:

{{< highlight console >}}
Host caipirinha
    Hostname 65.123.123.123
{{< / highlight >}}

And here's the inventory I called "hosts":

{{< highlight console >}}
[servers]
caipirinha
{{< / highlight >}}

When all the above is complete we can try to connect to the server.  
Since our VPS provider already uploaded the root key we connect as a root:
{{< highlight console >}}
$ ansible all -i hosts -u root -m ping
caipirinha | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
{{< / highlight >}}

What this command does is to use "hosts" file as an inventory, "root" as a user and "ping" as a module.  
"all" means that we execute the module on all entries in the inventory.  

For this example, instead of "all" one can use "servers" or "caipirinha" too.

By the way, "ping" might be confusing at first because what it really does is to perform a test SSH connection, not to send ICMP packet.

## Creating a playbook
  
Let's start with a file named playbook.yml.

We begin with the header: a name of our playbook, hosts it targets, and become directive which elevates the privileges (to root, by default).

{{< highlight yaml "linenos=table,linenostart=1" >}}
---
- name: Create user
  hosts: servers
  become: true
{{< / highlight >}}

We define a series of variables that will be used along the use of playbook.  
"username" is the name of the user that will be created on the server.  
"local_" prefixed variables will be used for generating SSH key pairs on our machine.  

[Lookup plugin](https://docs.ansible.com/ansible/latest/user_guide/playbooks_lookups.html) is used to access data from sources outside of the playbook like environment or filesystem.  

{{< highlight yaml "linenos=table,linenostart=5" >}}
vars:
  username: "seblw"
  local_home_path: "{{ lookup('env','HOME') }}"
  local_key_path:  "{{ local_home_path }}/.ssh/ \
      {{ username }}_{{ inventory_hostname }}"
  local_pubkey_path: "{{ local_key_path }}.pub"
  local_pubkey_file: "{{ lookup('file', \ 
    local_pubkey_path) }}"
{{< / highlight >}}

The next step is to make sure ["wheel" group](https://en.wikipedia.org/wiki/Wheel_(computing)) exists and let group members use sudo command without a password.

{{< highlight yaml "linenos=table,linenostart=12" >}}
tasks:
  - name: ensure 'wheel' group
    group:
      name: wheel
      state: present

  - name: allow 'wheel' group to have passwordless sudo
    lineinfile:
      path: /etc/sudoers
      state: present
      regexp: '^%wheel'
      line: '%wheel ALL=(ALL) NOPASSWD: ALL'
      validate: '/usr/sbin/visudo -cf %s'
{{< / highlight >}}

Here's the interesting piece. We use ["local_action" module](https://docs.ansible.com/ansible/latest/user_guide/playbooks_delegation.html) which let us act as the host computer i.e. the machine we run ansible command from.  
Since we've run ansible with `-u root` flag we need to switch back to the user we logged to our host machine - we get the username from `$USER` env. variable).

Later on, we generate SSH keypair in a location pointed by "local_key_path" variable, for this example, it's "~/.ssh/seblw_caipirinha".

{{< highlight yaml "linenos=table,linenostart=26" >}}
- name: generate an OpenSSH keypair on localhost
  become_user: "{{ lookup('env', 'USER') }}"
  local_action:
    module: openssh_keypair
    path: "{{ local_key_path }}"
    type: ed25519
{{< / highlight >}}

We then switch back to the remote machine and finally create a new user there which belongs to the "wheel" group and set the authorized key to the one we generated in the previous step.

{{< highlight yaml "linenos=table,linenostart=33" >}}
- name: create user '{{ username }}'
  user:
    name: "{{ username }}"
    state: present
    groups: ["wheel"]
    append: yes
    shell: /bin/bash

- name: set authorized key for user '{{ username }}'
  authorized_key:
    user: "{{ username }}"
    state: present
    key: "{{ local_pubkey_file }}"
{{< / highlight >}}

We also do not forget about security tweaks to SSH server configuration - disabling password authentication and (optionally) root login.  
Mind the notify directive, it calls a "handler".  

[Handler](https://docs.ansible.com/ansible/latest/user_guide/playbooks_handlers.html) is Ansible concept that allows us to run some predefined task on every state change, i.e. call a task to restart service after each configuration update.  

In our example, we trigger SSH server restart.

{{< highlight yaml "linenos=table,linenostart=47" >}}
- name: disable password authentication
  lineinfile:
    path: /etc/ssh/sshd_config
    state: present
    regexp: '^#?PasswordAuthentication'
    line: 'PasswordAuthentication no'
    validate: '/usr/sbin/sshd -T -f %s'
  notify: restart sshd

- name: disable root login
  lineinfile:
    path: /etc/ssh/sshd_config
    state: present
    regexp: '^#?PermitRootLogin'
    line: 'PermitRootLogin no'
    validate: '/usr/sbin/sshd -T -f %s'
  notify: restart sshd
{{< / highlight >}}

And at the end of the playbook, we define the handler.

{{< highlight yaml "linenos=table,linenostart=65" >}}
handlers:
  - name: restart sshd
    service:
      name: sshd
      state: restarted
{{< / highlight >}}


## Running the playbook

Now we are ready to run the playbook.  

{{< highlight console >}}
$ ansible-playbook -i hosts -u root playbook.yml
PLAY [Create user] ****************************************************

TASK [Gathering Facts] ************************************************
[...]
{{< / highlight >}}

When it is finished we can finally log in using the SSH key and username.  

{{< highlight console >}}
$ ssh -i ~/.ssh/seblw_caipirinha seblw@caipirinha
Welcome to Ubuntu 22.04.1 LTS (GNU/Linux 5.15.0-40-generic x86_64)
[...]
{{< / highlight >}}

Success!  


PS. Keeping proper indentation in YAML files is left as an exercise for the reader ;)