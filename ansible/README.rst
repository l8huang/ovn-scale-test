Ansible and Docker OVN Emulation Guide
======================================

Overview
--------

Deploy an OVN emulation environment using Docker container and ansible

**why using Docker?**

Docker allows deploying OVN emulation in a fast and consistent fashion.
Compiling OVS/OVN source is done once in one docker host, rather than
repetitively in every host. The same docker image will be distributed to all the
physical hosts by ansible.


Host machine requirements
-------------------------

The recommended emulation target requirements:

- 1 deploy node to run ansible and Rally workload
- 1 OVN database node
- 2 OVN chassis host to run emulated OVN chassis container
- 1 host to run rally container (could be the same as the OVN database node)

.. image:: ovn-emulation-deployment.png
   :alt: OVN emulation deployment

  
Installing Dependencies
-----------------------

The deploy node needs ansible. Docker and docker-py are required on the other nodes.

::

    pip install -U docker-py

Building OVN test Container Images
----------------------------------

To build containers, type

::

    cd ansible/docker
    make

These commands build two container images: ovn-scale-test-ovn and
ovn-scale-test-rally.

If you do not like the name, you can edit ansible/docker/Makefile to change the
image name.

Alternatively, there is a pre-built image in docker hub. To use it, run

::

    docker pull huikang/ovn-scale-test-ovn
    docker pull huikang/ovn-scale-test-rally

The remaining of this guide uses OVN-SCALE-TEST as the image name. You need to
change the name in your deployment.


Setup the emulation environment
-------------------------------

Add hosts to the ansible inventory file

::

    ansible/inventory/ovn-hosts

Start by editing ansible/group_vars/all.yml to fit your testbed.

For example, to define the total number of emulated chasis in the network:

::

    ovn_db_image: "huikang/ovn-scale-test"
    ovn_chassis_image: "huikang/ovn-scale-test"
    ovn_number_chassis: 10

During deployment, these chassis will be evenly distributed on the emulation
hosts, which are defined in the inventory file.

Deploying OVN Emulation
-----------------------

Run the ansible playbook

::

    ansible-playbook  -i ansible/inventory/ovn-hosts ansible/site.yml -e action=deploy

The above command deploys ovn-database and the emulated chassis. On the rally
node, an ovn-rally container is also launched as well as the deployment file and
workload files.

The fastest way during evaluation to re-deployment is to remove the OVN
containers and re-deploy.

To clean up the existing emulation deployment,

::

    ansible-playbook  -i ansible/inventory/ovn-hosts ansible/site.yml -e action=clean


Registerrinng with Rally
------------------------

The ansible playbook deploys both the emulated chassises, the rally container,
along with rally deployment file. The rally deployment file is used to register
the ovn emulation environment in the rally database, like to [1].

::

   docker exec -it ovn-rally bash


The following commands are run in the ovn-rally container

::

   (Generate ssh keys of the container and copy the keys to the other host
   listed in the inventory file)

   rally-ovs deployment create --file /root/rally-ovn/ovn-multihost-deployment.json --name ovn-multihost


The above command only registers the ovn emulation environment in the Rally
database.

To verify the deployment, in the ovn-rally container, type

::

   rally-ovs deployment list

   +--------------------------------------+----------------------------+---------------+------------------+--------+
   | uuid                                 | created_at                 | name          | status           | active |
   +--------------------------------------+----------------------------+---------------+------------------+--------+
   | a8d85fb4-c4ef-471b-ba11-cdb8885867d7 | 2016-05-02 16:47:34.278482 | ovn-multihost | deploy->finished | *      |
   +--------------------------------------+----------------------------+---------------+------------------+--------+

   rally-ovs deployment config


**TODO** Register emulated sandbox in the database

Rnning Rally Workloads
----------------------

**TODO**

References
----------
[1] http://rally.readthedocs.io/en/stable/tutorial/step_1_setting_up_env_and_running_benchmark_from_samples.html#registering-an-openstack-deployment-in-rally
