CloudStack Cloud Guide
======================

.. _introduction:

Introduction
````````````
The purpose of this section is to explain how to put Ansible modules together to use Ansible in a CloudStack context. You will find more usage examples in the details section of each module.

Ansible contains a number of extra modules for interacting with CloudStack based clouds. All modules support check mode and are designed to use idempotence and have been created, tested and are maintained by the community.

.. note:: Some of the modules will require domain admin or root admin privileges.

Prerequisites
`````````````
Prerequisites for using the CloudStack modules are minimal. In addition to ansible itself, all of the modules require the python library ``cs`` https://pypi.python.org/pypi/cs.

You'll need this Python module installed on the execution host, usually your workstation.

.. code-block:: bash

    $ pip install cs

.. note:: cs also includes a command line interface for ad-hoc ineraction with the CloudStack API e.g. ``$ cs listVirtualMachines state=Running``.

Credentials File
````````````````
You can pass credentials and the endpoint of your cloud as module arguments, however in most cases it is a far less work to store your credentials in the cloudstack.ini file.

The python library cs looks for the credentials file in the following order (last one wins):

* A ``.cloudstack.ini`` (note the dot) file in the home directory.
* A ``CLOUDSTACK_CONFIG`` environment variable pointing to an .ini file.
* A ``cloudstack.ini`` (without the dot) file in the current working directory, same directory as your playbooks are located.

The structure of the ini file must look like this:

.. code-block:: bash

    $ cat $HOME/.cloudstack.ini
    [cloudstack]
    endpoint = https://cloud.example.com/client/api
    key = api key
    secret = api secret

.. Note:: The section ``[cloudstack]`` is the default section. ``CLOUDSTACK_REGION`` environment variable can be used to define the default region.

Regions
```````
If you use more than one CloudStack region, you can define as many sections as you want and name them as you like, e.g.:

.. code-block:: bash

    $ cat $HOME/.cloudstack.ini
    [exoscale]
    endpoint = https://api.exoscale.ch/compute
    key = api key
    secret = api secret

    [exmaple_cloud_one]
    endpoint = https://cloud-one.example.com/client/api
    key = api key
    secret = api secret

    [exmaple_cloud_two]
    endpoint = https://cloud-two.example.com/client/api
    key = api key
    secret = api secret

.. Hint:: Sections can also be used to for login into the same region using different accounts.

By passing the argument ``api_region`` with the CloudStack modules, the region wanted will be selected.

.. code-block:: yaml

    - name: ensure my ssh pubkey exists on all CloudStack regions
      local_action: cs_sshkeypair
        name: my-ssh-key
        public_key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"
        api_region: "{{ item }}"
        with_items:
          - exoscale
          - exmaple_cloud_one
          - exmaple_cloud_two

Use Cases
`````````
The following should give you some ideas how to use the modules to provision VMs to the cloud. As always, there isn't only one way to do it. But as always: keep it simple for the beginning is always a good start.

Use Case: Provisioning in a Advanced Networking CloudStack setup
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
Our CloudStack cloud has an advanced networking setup, we would like to provision web servers, which get a static NAT and open firewall ports 80 and 443. Further we provision database servers, to which we do not give any access to. For accessing the VMs by SSH we use a SSH jump host.

This is how our inventory looks like:

.. code-block:: ini

    [cloud-vm:children]
    webserver
    db-server
    jumphost

    [webserver]
    web-01.example.com  public_ip=1.2.3.4
    web-02.example.com  public_ip=1.2.3.5

    [db-server]
    db-01.example.com
    db-02.example.com

    [jumphost]
    jump.example.com  public_ip=1.2.3.6

As you can see, the public IPs for our web servers and jumphost has been assigned as variable ``public_ip`` directly in the inventory.

The configure the jumphost, web servers and database servers, we use ``group_vars``. The ``group_vars`` directory contains 4 files for configuration of the groups: cloud-vm, jumphost, webserver and db-server. The cloud-vm is there for specifing the defaults of our cloud infrastructure.

.. code-block:: yaml

    # file: group_vars/cloud-vm
    ---
    cs_offering: Small
    cs_firewall: []

Our database servers should get more CPU and RAM, so we define to use a ``Large`` offering for them.

.. code-block:: yaml

    # file: group_vars/db-server
    ---
    cs_offering: Large

The web servers should get a ``Small`` offering as we would scale them horizontaly, which is also our default offering.

.. code-block:: yaml

    # file: group_vars/webserver
    ---
    cs_firewall:
      - { port: 80 }
      - { port: 443 }

Further we provision a jump host which has only port 22 opened for accessing the VMs from our office IPv4 network.

.. code-block:: yaml

    # file: group_vars/jumphost
    ---
    cs_firewall:
      - { port: 22, cidr: "17.17.17.0/24" }

Now to the fun part. We create a playbook to create our infrastructure we call it ``infra.yml``:

.. code-block:: yaml

    # file: infra.yaml
    ---
    - name: provision our VMs
      hosts: cloud-vm
      connection: local
      tasks:
        - name: ensure VMs are created and running
          cs_instance:
            name: "{{ inventory_hostname_short }}"
            template: Linux Debian 7 64-bit 20GB Disk
            service_offering: "{{ cs_offering }}"
            state: running

        - name: ensure firewall ports opened
          cs_firewall:
            ip_address: {{ public_ip }}
            port: {{ item.port }}
            cidr: "{{ item.cidr | default('0.0.0.0/0') }}"
          with_items: cs_firewall
          when: public_ip is defined

        - name: ensure static NATs
          cs_staticnat: vm="{{ inventory_hostname_short }}" ip_address="{{ public_ip }}"
          when: public_ip is defined

In the above play, we use the group ``cloud-vm`` to handle all VMs in the cloud but use ``connetion=local`` because we want the modules to be executed locally.

Note that for some modules, e.g. ``cs_sshkeypair`` you usually want this to be executed only once, not for every VM. Therefore you would make a separate play for this targeting localhost.

.. code-block:: yaml

    - name: configure ssh keys
      hosts: localhost
      connection: local
      tasks:
      - name: ensure my ssh pubkey exists
        cs_sshkeypair: name=my_key public_key="{{ lookup('file', '~/.ssh/id_rsa.pub') }}"


Use Case: Provisioning on a Basic Networking CloudStack setup
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

A basic networking CloudStack setup is slightly different: Every VM gets a public IP directly assigned and security groups are used for access restriction policy.

This is how our inventory looks like:

.. code-block:: ini

    [cloud-vm:children]
    webserver

    [webserver]
    web-01.example.com
    web-02.example.com

The default for your VMs looks like this:

.. code-block:: yaml

    # file: group_vars/cloud-vm
    ---
    cs_offering: Small
    cs_securitygroups: [ 'default']

Our webserver will also be in security group ``web``:

.. code-block:: yaml

    # file: group_vars/webserver
    ---
    cs_securitygroups: [ 'default', 'web' ]

The playbook looks like the following:

.. code-block:: yaml

    # file: infra.yaml
    ---
    - name: cloud base setup
      hosts: localhost
      connection: local
      tasks:
      - name: upload ssh public key
        cs_sshkeypair:
          name: defaultkey
          public_key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"

      - name: ensure security groups exist
        cs_securitygroup:
          name: "{{ item }}"
        with_items:
          - default
          - web

      - name: add inbound SSH to security group default
        cs_securitygroup_rule:
          security_group: default
          start_port: "{{ item }}"
          end_port: "{{ item }}"
        with_items:
          - 22

      - name: add inbound TCP rules to security group web
        cs_securitygroup_rule:
          security_group: web
          start_port: "{{ item }}"
          end_port: "{{ item }}"
        with_items:
          - 80
          - 443

    - name: install VMs in the cloud
      hosts: cloud-vm
      connection: local
      tasks:
      - name: create and run VMs on cloudstack
        cs_instance:
          name: "{{ inventory_hostname_short }}"
          template: Linux Debian 7 64-bit 20GB Disk
          service_offering: "{{ cs_offering }}"
          security_groups: "{{ cs_securitygroups }}"
          ssh_key: defaultkey
          state: Running
        register: vm

      - name: show VM IP
        debug: msg="VM {{ inventory_hostname }} {{ vm.default_ip }}"

      - name: assing IP to the inventory
        set_fact: ansible_ssh_host={{ vm.default_ip }}

      - name: waiting for SSH to come up
        wait_for: port=22 host={{ vm.default_ip }} delay=5

In the first play we setup the security groups, in the second play the VMs will created be assigned to these groups. Further you see, that we assign the public IP returned from the modules to the host inventory. This is needed as we do not know the IPs we will get in advance. In a next step you would configure the DNS servers with these IPs for accassing the VMs with their DNS name.

In the last task we wait for SSH to be accessible, so any later play would be able to access the VM by SSH without failure.
