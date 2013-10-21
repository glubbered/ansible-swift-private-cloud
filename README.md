Ansible Swift Install
=====================

This repo contains a set of Ansible playbooks to fire up the [Swift Private Cloud](https://github.com/rcbops-cookbooks/swift-private-cloud) Chef cookbooks in the reference implementation.

Quick Install
=============

Install Ansible and PyRax
-------------------------

As of this writing, pyrax 1.5.3 is broken when creating cloud networks. The working branch works until 1.5.4 is out.

    $ [sudo] pip install ansible
    $ [sudo] pip install git+git://github.com/rackspace/pyrax@working

Provision Servers
-----------------

This repo includes a playbook to provision a set of Rackspace Cloud Servers. It will create:

* Isolated Network ( "swift_management", "192.168.122.0/24" )
* Chef Server x 1
* Management Server x 1
* Storage Servers x 3
* Proxy Servers x 2

Each server will belong to PublicNet, ServiceNet and the swift_management network.

First, ensure you have a credentials file configured with your `username` and `api_key`:

    # file: ~/.rackspace_cloud_credentials
    [rackspace_cloud]
    username = yourusername
    api_key = yourapikey

We're assuming you have an `~/.ssh/id_rsa.pub` and it will be uploaded to the root account.

Then run the `provision/rackspace.yml` playbook:

    $ ansible-playbook provision/rackspace.yml

When this completes, it will create a hosts inventory file called `rackapace`:

    [management-server]
    192.0.2.2

    [storage-servers]
    192.0.2.3
    192.0.2.4
    192.0.2.5

    [proxy-servers]
    192.0.2.6
    192.0.2.7

    [chef-server]
    192.0.2.1

    [chef-clients:children]
    management-server
    proxy-servers
    storage-servers

Once complete, ensure you can ssh to each instance properly:

    $ ansible all -i rackspace -m command -a "whoami"

Additional provision playbooks will be created on an as needed basis.

You can also create your own hosts inventory file and point it to already existing servers as long as the following are true:

* the group names match the above names
* you've already configured ssh public key access for the root account

Configure Servers
-----------------

To configure the servers in the inventory file, run the `site.yml`:

    $ ansible-playbook -i rackspace site.yml

This will:

* Install Chef Server on any servers in the `chef-server` group.
* Install Chef Client on any servers in the `chef-clients` group, including configuring and registering nodes to talke to the chef server.
* Run the `spc-starter-controller` chef role on server in the `management-server` group.
* Run the `spc-starter-storage` chef role on the servers in the `storage-servers` group.
* Run the `spc-starter-proxy` chef role on the servers in the `proxy-severs` group.
* Re-run the `spc-starter-controller` chef role on server in the `management-server` group to gather all the other nodes.
* Re-run the `spc-starter-proxy` chef role on the proxy servers again to propgate all memcached servers addresses.

While running `site.yml` runs everything, you can also run them role by role:

    $ ansible-playbook -i rackspace chef-server.yml
    $ ansible-playbook -i rackspace chef-clients.yml
    $ ansible-playbook -i rackspace management-server.yml
    ...

...and even server by server:

    $ ansible-playbook -i rackspace storage-servers.yml --limit 192.0.2.3

Caveat Executor
===============

This is just a first pass. Batteries not included. We are making some assumptions:

* The isolated network being created matches the swift environment file: `192.168.122/0/24`
* If you use your own servers instead, you have one network interface matching `192.168.122.0/24`
* You can root ssh to these machines using ssh public keys. (This can probably be changed via `ansible.cfg` or command line args)
* You're running CentOS 6.4. Ubuntu 12.04 needs to be cussed out.

Many things are hard coded but will become more flexible in the future as they slowly move from cookbooks into playbooks. Not responsible if your dog catches fire or your significant other files for divorce.

Repository Structure
====================

This repository is configured close to the [Ansible Best Practices](http://www.ansibleworks.com/docs/playbooks_best_practices.html#directory-layout) with the exception of the `provision` folder. This may become a role in the future, but I didn't want to intermingle it and cause confusion right now.

The roles in this repo mimic the top level recipes in the Chef cookbooks. If we move forward with Ansible, we can just move things from the recipes into the exists top level roles without having to change the site/role playbooks.
