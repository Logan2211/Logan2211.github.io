---
layout: post
title:  "OpenStack-Ansible Overlay"
date:   2016-07-31 13:10:00 -0500
categories: openstack ansible devops deployment automation
---

## What is Overlay and why should I use it?

One of the major advantages I gain by operating my cloud using
[OpenStack-Ansible](http://docs.openstack.org/developer/openstack-ansible/index.html)
\(OSA\) as the deployment system is the ability to easily adapt my
[OpenStack](https://www.openstack.org) cloud for a variety of environments.
However, when maintaining these deployments across upgrades and over time, it
can become a challenge to merge user space configurations with changes to the
upstream OpenStack-Ansible repository.

What I needed was a way to add additional services and playbooks and deploy
various layers of user-space configuration to my OpenStack-Ansible setup without
modifying the base OSA repository.

After experimenting with several methods, I found the [rpc-openstack](https://github.com/rcbops/rpc-openstack)
repository and built an OpenStack-Ansible overlay inspired by rpc-openstack's
structure.

## Where can I find Overlay?

The openstack-ansible-overlay repository is located on [GitHub](https://github.com/Logan2211/openstack-ansible-overlay).
Included in the repository are scripts used to perform the configuration merging
and deployment of overlay playbooks.

## How do I use Overlay?

The example overlay repository contains an example of an NTP role added to the
OpenStack-Ansible environment. To test out the example Overlay, we can build
an AIO as follows. The AIO build is almost identical to the build created by
following the [OpenStack-Ansible Quick Start](http://docs.openstack.org/developer/openstack-ansible/developer-docs/quickstart-aio.html)
guide.

{% highlight bash %}
# First clone the overlay repository
# The --recursive option will initialize the openstack-ansible submodule
# inside the repository.
git clone --recursive https://github.com/Logan2211/openstack-ansible-overlay
cd openstack-ansible-overlay

# Update the submodule to the HEAD of the openstack-ansible master branch
git submodule update --remote

# Run the deployment to build the AIO
# UPDATE_ANSIBLE is used to run the bootstrap-ansible.sh script included with OpenStack-Ansible
# DEPLOY_AIO triggers the AIO bootstrap role
sudo UPDATE_ANSIBLE=yes DEPLOY_AIO=yes scripts/deploy.sh
{% endhighlight %}

## Customization

Directory structure of the example overlay repository:
{% highlight plaintext %}
overlay
├── env
│   ├── env.d
│   ├── group_vars
│   └── openstack_deploy
│       ├── user_ntp.yml
│       └── user_settings.yml
├── local
│   ├── env.d
│   ├── group_vars
│   └── openstack_deploy
│       └── examples
│           ├── user_ntp.yml
│           └── user_settings.yml
└── playbooks
    ├── inventory
    │   └── group_vars
    └── ntp-install.yml
{% endhighlight %}

# Configuration Overrides

Configuration overrides can be placed in `env/openstack_deploy/user_*.yml` or
`local/openstack_deploy/user_*.yml` files.

After making changes to these files, run `scripts/deploy.sh` again. To rebuild
`/etc/openstack_deploy` without executing the playbooks, run
`scripts/update-openstack-deploy.sh`.

When these scripts are executed, the configuration files are merged from the
following directories:

1. `openstack-ansible/etc/openstack_deploy`
2. `overlay/env/openstack_deploy`
3. `overlay/local/openstack_deploy`

If, for example, a file named `user_variables.yml` exists in all 3 directories
above, then the one included in the openstack-ansible repository will be used as
the base, and the overlay/env overrides and overlay/local overrides will be
merged on top in that order.

Files do not need to exist in all 3 layers to be
merged. If a file only exists in `env` and `local`, then the `env` file will be
treated as the default, and the `local` file will be merged on top.

Using this format it is possible to have a base
set of organizational configuration variables defined in `env`, while variables
for a specific region can be added in `local`.

# Group Variable Overrides

Group variables can be merged the same way as global configuration variables. To
configure group variable overrides, drop configurations in `env/group_vars` or
`local/group_vars`, then update the merged configurations using
`scripts/update-openstack-deploy.sh`.

Group variables are merged in the following order:

1. `openstack-ansible/playbooks/inventory/group_vars`
2. `overlay/env/group_vars`
3. `overlay/local/group_vars`

# Environment Overrides

Environment overrides are merged using files from `env/env.d` and `local/env.d`
in a similar fashion to the vars files above. However, since OSA includes
environment override support as of Newton, env.d overrides do not need to be
merged with the base environment in the OSA tree. Due to this, environment
overrides are merged in the following order:

1. `overlay/env/env.d`
2. `overlay/local/env.d`

# A note about configuration files

Files in the following directories are dynamically generated and should not be
modified manually.

- `/etc/openstack_deploy`
- `overlay/playbooks/inventory/env.d`
- `overlay/playbooks/inventory/group_vars`

### Extending playbooks/roles

Additional roles can be added to the `ansible-role-requirements.yml` file, and
playbooks can be added to `overlay/playbooks`.

Then, deployment scripts can be added to `scripts/` or `scripts/deploy.sh` can
be modified to execute the additional playbooks.

## Usage Notes

The example repository is meant to be forked and extended further. It was
created to demonstrate extension of OpenStack-Ansible's core feature set and
facilitate the construction of maintainable configurations for clouds deployed
by OpenStack-Ansible.
