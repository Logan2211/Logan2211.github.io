---
layout: post
title:  "Integrating OpenStack-Ansible and ceph-ansible"
date:   2016-08-02 15:05:00 -0500
categories: openstack ansible devops deployment automation ceph storage
---

## Overview

[OpenStack-Ansible](http://docs.openstack.org/developer/openstack-ansible/index.html)
supports deploying an [OpenStack](https://www.openstack.org) cloud configured to
use an existing [Ceph](https://ceph.com) storage cluster, however it does not
support the actual deployment of the Ceph cluster it consumes.

This guide explains how to integrate the
[ceph-ansible](https://github.com/ceph/ceph-ansible) playbooks with
OpenStack-Ansible to manage and deploy both systems together as a single unit.

## Prerequisite

This guide will use the
[Openstack-Ansible Overlay]({% post_url 2016-07-31-openstack-ansible-overlay %})
framework as the basis for integrating the two projects.

## Quick Start

I have prepared a
[sample branch](https://github.com/logan2211/openstack-ansible-overlay/tree/ceph)
using the Overlay to add ceph-ansible to the OpenStack-Ansible deployment.

To get started with the sample branch, build an AIO as follows:

{% highlight bash %}
# First clone the overlay repository
# The --recursive option will initialize the openstack-ansible submodule
# inside the repository.
git clone -b ceph --recursive https://github.com/Logan2211/openstack-ansible-overlay
cd openstack-ansible-overlay

# Update the submodule to the HEAD of the openstack-ansible master branch
git submodule update --remote

# Run the deployment to build the AIO
# UPDATE_ANSIBLE is used to run the bootstrap-ansible.sh script included with OpenStack-Ansible
# DEPLOY_AIO triggers the AIO bootstrap role
sudo UPDATE_ANSIBLE=true DEPLOY_AIO=true scripts/deploy.sh
{% endhighlight %}

Once the AIO has finished building, you will have an environment which is
configured to use Ceph along with a working Ceph cluster inside the AIO.

## How it works

Integrating the ceph-ansible playbooks into the OpenStack-Ansible Overlay
configuration involves several configuration files.

* `ansible-role-requirements.yml`: Used to clone the ceph-ansible roles into
  /etc/ansible/roles
* `overlay/env/env.d/ceph.yml`: Impelments the ceph-mon and ceph-osd containers
  and groups.
* `overlay/env/openstack_deploy/user_ceph.yml`: The baseline configuration
  required to run the ceph-ansible roles.
* `overlay/env/openstack_deploy/user_settings.yml`: Configuration used to
  make OpenStack-Ansible use our Ceph cluster.
* `overlay/local/openstack_deploy/user_ceph.yml`: Automatically created when
  deploying an AIO using the example repo, this file will contain settings to
  bootstrap the Ceph cluster inside the AIO.
* `overlay/playbooks/ceph-install.yml`: Playbook used to run the ceph-ansible
  roles.
* `scripts/deploy.sh`: It executes the playbooks in a similar fashion to
  OpenStack-Ansible's `run-playbooks.sh`, however in this case it runs the
  ceph-install.yml playbook at the appropriate time during the OpenStack-Ansible
  deployment process.

# AIO Ceph Architecture

The AIO build included in the example repo will configure a single ceph-mon
container along with 3 OSDs on the host. The OSDs are provisioned using loopback
devices that can be found at `/openstack/ceph{1,2,3}.img`.

## Further Customization

For more detailed information about this example configuration, please review
the branch implementing the Ceph integration at
[Github](https://github.com/Logan2211/openstack-ansible-overlay/compare/ceph).

The build can be further customized by following the instructions for
[Overlay customizations]({% post_url 2016-07-31-openstack-ansible-overlay %}#customization).
ceph-ansible offers a variety of configuration options, which are outlined in
the [group_vars sample files](https://github.com/ceph/ceph-ansible/tree/master/group_vars).
