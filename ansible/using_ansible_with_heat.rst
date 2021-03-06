=======================
Using Ansible with Heat
=======================

Brief summary
=============

In this tutorial, we will show you how to leverage Ansible via Heat and software config to
bootstrap an instance with a fully configured Nginx server.

Pre-reading
===========

- You should prepare a bootstrapped image according to the `Bootstrapping Software Config
  <../boostrapping_software_config.rst>`_ tutorial as we will be making use of the image
  pre-configured with all the required agents for software config
- This tutorial borrows heavily from `An Ansible Tutorial <https://serversforhackers.com/an-ansible-tutorial>`_.
  Reading this guide will give you a good idea of what we'll be installing/configuring so
  you can focus on how we use Heat to integrate with Ansible rather than using Ansible
  itself.

Following Along
===============

You will probably want to clone this repository in order to easily follow along. Once
cloned, change to the ``ansible`` directory of the repository.

Otherwise, you may need to modify some of the commands to point to the correct locations
of various templates and/or environments. Full templates can always be found in the
``templates`` directory.

Basic Template
==============

As with all Heat templates, we start with the basic version and description sections:

.. code:: yaml

  heat_template_version: 2014-10-16

  description: |
    Deploy Nginx server with Ansible

For the parameters section, we'll define a single parameter that will tell Heat which
image to use for our server. This allows us some flexibility should the image name change
or we have several to choose from:

.. code:: yaml

  parameters:

  image:
    type: string

Resources
---------

Now for the working bits. First, we create a random password for accessing the server:

.. code:: yaml

  server_pw:
    type: OS::Heat::RandomString

Next, we specify the playbook that will install Nginx:

.. code:: yaml

  nginx_config:
    type: OS::Heat::SoftwareConfig
    properties:
      group: ansible
      config: |
        ---
        - name: Install and run Nginx
          connection: local
          hosts: localhost
          tasks:
           - name: Install Nginx
             apt: pkg=nginx state=installed update_cache=true
             notify:
              - Start Nginx
          handlers:
           - name: Start Nginx
             service: name=nginx state=started

We then use an ``OS::Heat::SoftwareDeployment`` to tell Heat we want to run the playbook
on our server (which we will define in a bit):

.. code:: yaml

  deploy_nginx:
    type: OS::Heat::SoftwareDeployment
    properties:
      signal_transport: TEMP_URL_SIGNAL
      config:
        get_resource: nginx_config
      server:
        get_resource: server

Finally, we'll define the server the playbook will run on:

.. code:: yaml

  server:
    type: OS::Nova::Server
    properties:
      image: { get_param: image }
      admin_pass: { get_attr: [ server_pw, value ] }
      flavor: 2 GB Performance
      software_config_transport: POLL_TEMP_URL
      user_data_format: SOFTWARE_CONFIG

Notice that we have to specify the ``user_data_format`` as "SOFTWARE_CONFIG" so that Heat
knows to set up the proper signal handling between it and the server. Its a good practice
to specify ``software_config_transport`` and while "POLL_TEMP_URL" is the only value
supported on the Rackspace Cloud, it should also be the default for Cloud Orchestration
and can be safely omitted.

Outputs
-------

The outputs defined in this template give us ready access to the results of the deployment
and show off how software config makes it easier to see the state of your configuration,
the results, and any errors or output it may have generated without having to remotely
log into your servers and search through logs. The ``description`` property of these
outputs tell you what each represents.

.. code:: yaml

  outputs:
    stdout:
      description: Ansible Output
      value:
        get_attr: [ deploy_nginx, deploy_stdout ]
    stderr:
      description: Ansible Error Output
      value:
        get_attr: [ deploy_nginx, deploy_stderr ]
    status_code:
      description: Exit Code
      value:
        get_attr: [ deploy_nginx, deploy_status_code ]
    server_ip:
      description: Server IP Address
      value:
        get_attr: [ server, accessIPv4 ]
    server_password:
      description: Server Password
      value:
        get_attr: [ server_pw, value ]

Deploy the Basic Template
=========================

Before you deploy, you'll need to have created an image that already has the needed
agents for software config. The `Bootstrapping Software Config
<../bootstrapping_software_config.rst>`_ walks you through it. Alternatively, you can use
the information in that an previous tutorials to add the appropriate bootstrapping to this
template.

To deploy this template, simply issue the standard command:

.. code:: example

  heat stack-create -f templates/software_config_ansible.yaml -P "image=Ubuntu 14.04 LTS (HEAT)" my_nginx_simple

Once the stack is ``CREATE_COMPLETE``, you can visit your new Nginx homepage by checking
the stack output for the ip and entering that into your web browser:

.. code:: example

  heat output-show my_nginx_simple server_ip
  
You can also check the results of the playbook by checking the other outputs:

.. code:: example

  heat output-show my_nginx_simple status_code  # Ansible return code
  heat output-show my_nginx_simple stdout       # Ansible output
  heat output-show my_nginx_simple stderr       # Error details (if any; should be empty)

Advanced Template with Roles
============================

While the basic template gives a good idea of how Heat integrates with Ansible, lets look
at a slightly more advanced usage leveraging Ansible roles. We'll tweak the previous
template a bit, so lets make a copy and call it "software_config_ansible_role.yaml

The role and its components can be found in this repository under the ``roles`` directory.

New Resources
-------------

We'll add two new resources to pull down the role we want to use and put it in a place
Ansible can get to it:

.. code:: yaml

  pull_role_config:
    type: OS::Heat::SoftwareConfig
    properties:
      group: ungrouped
      config: |
        #!/bin/bash
        git clone https://github.com/rackerlabs/rs-heat-docs.git
        cp -r rs-heat-docs/ansible/roles/nginx /etc/ansible/roles/

This is a simple script that clones this repository and copies the role to the right
place.

We'll also deploy that script to the server:

.. code:: yaml

  deploy_role:
    type: OS::Heat::SoftwareDeployment
    properties:
      signal_transport: TEMP_URL_SIGNAL
      config:
        get_resource: pull_role_config
      server:
        get_resource: server

Modify Playbook
---------------

Since we're using roles to do all of the heavy lifting, we'll modify our ``nginx_config``
resource to simply apply the role:

.. code:: yaml

  nginx_config:
    type: OS::Heat::SoftwareConfig
    properties:
      group: ansible
      config: |
        ---
        - hosts: localhost
          roles:
          - nginx

We'll also need to modify the deployment of the playbook to depend on the ``deploy_role``
resource since we'll need the role installed before we can apply it:

.. code:: yaml

  deploy_nginx:
    type: OS::Heat::SoftwareDeployment
    depends_on: deploy_role
    properties:
      signal_transport: TEMP_URL_SIGNAL
      config:
        get_resource: nginx_config
      server:
        get_resource: server

Modify Outputs
--------------

Our script for pulling the role definition isn't terribly sophisticated. We aren't
capturing or writing any output, but we can examine the exit code of our script. We'll add
that to the ``outputs`` section so we can check it if we need to:

.. code:: yaml

  role_status_code:
    description: Exit Code returned from deploying the role to the server
    value:
      get_attr: [ deploy_role, deploy_status_code ]

Deploy the Advanced Template
============================

Deploying the new template is the same as above, we just change the template name:

.. code:: example

  heat stack-create -f templates/software_config_ansible_role.yaml -P "image=Ubuntu 14.04 LTS (HEAT)" my_nginx_role

We can also check outputs the same way by simply changing the stack name:

.. code:: example

  heat output-show my_nginx_role status_code      # Ansible return code
  heat output-show my_nginx_role stdout           # Ansible output
  heat output-show my_nginx_role stderr           # Error details (if any; should be empty)
  heat output-show my_nginx_role role_status_code # Exit code of the role script

Reference documentation
=======================

- `Ansible Tutorial (much of this guide is cribbed from here) <https://serversforhackers.com/an-ansible-tutorial>`_
- `Ansible Homepage <http://www.ansible.com/home>`_
