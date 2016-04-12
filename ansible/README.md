# Automated rollout via Ansible
## Prerequisites
1. Install Ansible 2 on your local machine and run all the playbooks from there
1. Clone this repository to your local machine
1. Download the following binary files and put them either into the local ansible/binary directory or ensure they are already present on the layer1 host and configure the host_vars/layer1.yml paramter "layer1_binary_dir":
  - [RHEL-OSP overcloud binaries](https://access.redhat.com/downloads/content/191/ver=7/rhel---7/7/x86_64/product-software)
    - Overcloud image
    - Deployment ramdisk
    - Discovery ramdisk
  - [RHEL 7 binary DVD](https://access.redhat.com/downloads/content/69/ver=/rhel---7/7.2/x86_64/product-software)
1. Download the manifest for your Organization for the Satellite and copy the manifest-zip file to the local ansible/binary directory and rename it to manifest.zip
1. Change into the ansible directory, and copy or create the necessary ssh key pairs in the binary directory (the first one is used for the communication between the RHOSP-director and the layer1 host, the second to connect to the layer1 host from the outside):
  - $ ssh-keygen -t rsa -f binary/undercloud
  - $ ssh-keygen -t rsa -f binary/hailstorm
1. Adapt the host_vars/layer1.yml settings
  - rhel_iso_img: to the name of the RHEL 7 binary DVD ISO image
  - ansible_host: to the ip address or DNS name of your layer1 host (which is prepared with a minimal RHEL install).  
  - If no ssh keys are available, set the ansible_ssh_pass parameter to the hosts root password (see [ansible documentation](http://docs.ansible.com/ansible/intro_inventory.html))
1. Adapt the host_vars/rhosp-director.yml settings:
  - deploy_ramdisk_image
  - discovery_ramdisk_image
  - overcloud_image
1. Adapt the host_vars/satellite.yml settings
  - ntpserver
  - poolid for the Satellite repository

## Running the playbook
Run all commands on your laptop from the ansible directory. Since the server might reboot when the playbook executes, running the playbook on the server is discouraged.

### Setting up the environment
Everything:
```
$ ansible-playbook -i hosts create.yml
```
Only layer1 and OpenStack (see create.yml source code for available tags):
```
$ ansible-playbook -i hosts create.yml --tags layer1,rhosp
```
### Tearing down the environment
Everything:
```
$ ansible-playbook -i hosts destroy.yml
```
Only OpenStack:
```
$ ansible-playbook -i hosts destroy.yml --tags rhosp
```

## Understanding the playbooks
The playbooks (create.yml, destroy.yml - both in this directory) describe a sequence of "roles" to be applied to groups of machines. Each role in itself is a sequence of idempotent steps, i.e. they can be run more than one time without affecting the result.

### Inventory

The groups of machines referenced in the playbooks are defined in the [inventory](http://docs.ansible.com/ansible/intro_inventory.html) file "hosts" in this directory. These groups could for example be the virtual machines that make up OpenStack, RHEV, etc... Unless you are defining or changing a group of (virtual) machines, it is unlikely that you will need to change this file.

If a group consists only of a single machine (e.g. layer1), it has the same name as the machine itself. This might be confusing at first because it becomes clear only by knowing Ansible conventions that something refers to a single host versus a group (Hint: it is usually a group).

All playbooks assume that there is a single layer1 host in the layer1 group - they will fail / yield unexpected results if there are more than one layer1 host in the group.

### Configuration variables

For each (virtual) machine, a set of configuration properties is defined as follows: first, Ansible determines all groups a machine belongs to (based on the data in the inventory file). It then reads all corresponding properties files in the group_vars directory. Then, it reads the machine-specific configuration properties from the corresponding properties file in the host_vars directory, potentially overwriting a group property.

This approach allows to define common configurations on a group level with the ability to specify machine-specific configuration on a host level. Being written in YML, the properties can be complex data structures, i.e. structs and lists. You can name the properties (almost) any way you like, but be cautious prepending them with ansible_ - [such properties](http://docs.ansible.com/ansible/intro_inventory.html#list-of-behavioral-inventory-parameters) may be interpreted by the Ansible runtime itself. For example, ansible_host specifies the IP address that Ansible uses to connect to that specific machine via ssh.

### Roles

Roles are an Ansible concept which allows related tasks, templates, etc... to be grouped together, each in a separate subdirectory of roles. The entry point for each role is its tasks/main.yml file.

Ansible does not have a native concept of different actions such as "create the environment" or "tear down the environment" or "ensure that everything is reset after a demo" - all it does is run the tasks in the playbook / roles. This is why the roles' tasks/main.yml file is typically written to expect a variable named "mode" to be set to a specific verb such as "create" or "destroy". It then uses this variable to decide which actions to run, which are usually included as separate yml files.

The following roles exist:
- **common**: not a role in its own sense, rather a place where tasks, handlers or templates which are used in multiple roles can be stored and referenced.
- **layer1**: configures the layer1 host, network setup and services required by the layer2 VMs such as an export directory for kickstart files, NTP service, etc...
- **layer1_vms**: creates the virtual machines for an inventory group and installs a base RHEL via kickstart. Since the role is applied to the layer1 host, the name of the group is passed via variable name. This also means that all the variables/facts defined for the individual members of the group - i.e. the host that is to be instantiated - is not avaiable in the default scope. Most of the tasks iterate over the group members, so the host name is avialable as *item*. This means that you can access the actual host variables / facts via *hostvars[item].nameofvariable*.
- **layer2_rhel**: configures the base RHEL installed in the previous step - a great place to put common configuration actions:
  - Subscribes/Unsubscribes the VM
  - Attaches it to a pool
  - Performs an upgrade to the latest package versions
  - Installs additional packages
  - Enables or disables services
- **layer2_satellite**: configures Red Hat Satellite (runs katello installer, creates content views, activation keys, etc...)
- **layer2_rhosp_director**: configures Red Hat OpenStack Director (undercloud) and prepares "baremetal" nodes for installation of the RHOSP overcloud (runs for about 45 mins)
- **layer2_rhosp_overcloud**: deploys Red Hat OpenStack (overcloud) on the nodes prepared by director. This is still work in progress (runs for about 45 mins).

All other roles are not yet usable.

### Network Connectivity

In order to access the virtual machines created on layer2, a tunneling mechanism is used. Ansible connects to the layer1 host and from there tunnels to the virtual machines via a dedicated admin network. See group_vars/layer2.yml to see what this tunneling mechanism actually looks like.

#### Accessing layer2 hosts via browser

- Log into the layer1 host using the following command to establish a SOCKS proxy on localhost port 1080
  ```
  $ ssh -i binary/hailstorm -D 1080 root@<NAME_OF_IP_OF_THE_LAYER1_HOST>
  ```
- Configure your browser to use a SOCKS proxy on localhost port 1080

#### Accessing layer2 host consoles via VNC
To debug installation processes, a connection to the VM console might be required. This can be achieved roughly by the follwoing approach:
- ensure that the VMs host variables contain the following property
  ```
  graphics: vnc,listen=0.0.0.0,password=redhat01
  ```
- log into your layer1 host and use the following command to determine which port the VNC console actually runs on (you can probably also specify a fixed port number; check the virt-install documentation)
  ```
  # virsh dumpxml <vm_name>
  ```
- log into the layer1 host again to port-forward a local port to the console (the second port number needs to be changed to the port number from the XML dump).
  ```
  $ ssh -i binary/hailstorm -L 5901:localhost:5901 root@<NAME_OF_IP_OF_THE_LAYER1_HOST>
  ```
- connect your VNC viewer to localhost:5901