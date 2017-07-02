# Hyperledger Fabric Deployment

This ansible playbook project will accomplish the following tasks

 - Provision virtual machine nodes to participate in fabric network
 - Install necessary hyperledger dependent libraries and packages
 - Setup overlay network so containers can communicate cross multiple docker host
 - Install registrator and dns services so that containers can be referenced by name
 - Build hyperledger fabric artifacts
 - Run hyperledger fabric tests
 - Generate fabric network certificats, genesis blocks, transaction file etc
 - Push new or tagged fabric images onto all docker hosts
 - Deploy fabric network
 
## Status

In development

## Requirements

- [Install Ansible](http://docs.ansible.com/ansible/intro_installation.html)
- [Ubuntu 16.04 machines] (https://cloud-images.ubuntu.com/releases/16.04/)
- Clone this project into a directory.

If you will be using an Ubuntu system as Ansible controller, then you can
easily setup an environment by running the following script. If you have
other system as your Ansible controller, you can do similar steps to setup
the environment, the command may not be exact the same but the steps you
need to do should be identical.

    sudo apt-get update
    sudo apt-get install python-dev python-pip libssl-dev libffi-dev -y
    sudo pip install --upgrade pip
    sudo pip install six==1.10.0
    sudo pip install ansible==2.3.0.0
    git clone https://github.com/litong01/fabric-deploy.git

This project requires that you use Ansible version 2.3.0.0 or above


## Run the script to provision docker hosts.

This step is to provision a set of virtual servers from an OpenStack cloud.
Before you run the following command, you will need to either make a copy
of the vars/os.yml file or make changes to that file to reflect your cloud
settings. If you already have a set of servers (such as a set of VirtualBox
virtual machines), you can skip this step, by you will need to follow the
instructions below to create a runhosts file.

With the correct cloud environment settings in vars/os.yml, run the script
to provision a set of virtual machines::

    ansible-playbook -e "mode=apply env=os password=XXXXX" provcluster.yml


The above command will provision (prov is short for provision) a cluster of
virtual machines on your OpenStack cloud the environment defined in vars/os.yml
file. Replace xxxxx with your own password from your cloud provider. Replace
os with your own cloud environment file if you decided to create a new one.
This step produces a set of servers and an ansible host file named run/runhosts.


### Manually create a runhosts file with servers already available

If you already have a set of servers available that you wish to use, then you
can create a file by following the example below. And also make sure these
server's hostname get setup as XXXXX001, XXXXX002, etc and they can can see
each other by their hostnames. The XXXXX should be replaced with your own
perference which gets used in the later configuration. In this example, the
word "fabric" is used, but it can be anything that you prefer, make sure
they are consistent.

    cloud ansible_host=127.0.0.1 ansible_python_interpreter=python
    169.45.102.186 private_ip=10.0.10.246 public_ip=169.45.102.186 inter_name=fabric001
    169.45.102.187 private_ip=10.0.10.247 public_ip=169.45.102.187 inter_name=fabric002
    169.45.102.188 private_ip=10.0.10.248 public_ip=169.45.102.188 inter_name=fabric003

    [allnodes]
    169.45.102.186
    169.45.102.187
    169.45.102.188

    [etcdnodes]
    169.45.102.186
    169.45.102.187
    169.45.102.188

    [builders]
    169.45.102.186

The above file is a typical ansible host file. The cloud ansible_host should be your ansible
controller server, you should not change that line. All other lines in the file represent
a server, private_ip and public_ip are the concept for cloud, if your servers are not in
a cloud, then you can use the server's IP address for both private_ip and public_ip field,
but you can not remove these two fields. The inter_name is also important, you should name
the server sequentially and these names will be used in later configuration to allocate
hyperledger fabric components. Group allnodes should list all the servers other than the
ansible controller node. Group etcdnodes should list all the servers that you wish to install
etcd services on. Group builders should list all the servers that you wish to use to build
hyperledger fabric artifacts such as executables and docker images.

## Install all fabric dependencies and setup the overlay network for the docker hosts::

    ansible-playbook -i run/runhosts -e "mode=apply env=os" initcluster.yml

## Setup the fabric network::

    ansible-playbook -i run/runhosts -e "mode=apply env=bc1st" setupfabric.yml

The env value in the command indicates which fabric network configuration to use.
In above example, ansible looks for a file in vars directory named bc1st.yml,
you can create as many files in that directory to reflect your own fabric network.
Here is the bc1st.yml (short for block chain 1st network)::

    ---
    # The url to the fabric source repository
    GIT_URL: "http://gerrit.hyperledger.org/r/fabric"

    # The gerrit patch set reference, should be automatically set by gerrit
    GERRIT_REFSPEC: "refs/tags/v1.0.0-rc1"

    # This variable defines fabric network attributes
    fabric: {
      ssh_user: "ubuntu",
      network: {
        fabric001: {
          cas: ["ca.orga", "ca.orgb"],
   	      peers: ["leader@1stpeer.orga", "leader@1stpeer.orgb"],
          orderers: ["1storderer.orgc", "1storderer.orgd"],
          zookeepers: ["zookeeper1st"],
   	      kafkas: ["kafka1st"]
   	    },
   	    fabric002: {
   	      cas: ["ca.orgc", "ca.orgd"],
          peers: ["anchor@2ndpeer.orga", "anchor@2ndpeer.orgb"],
          orderers: ["2ndorderer.orgc", "2ndorderer.orgd"],
          zookeepers: ["zookeeper2nd"],
          kafkas: ["kafka2nd"]    
        },
        fabric003: {
          peers: ["worker@3rdpeer.orga", "worker@3rdpeer.orgb"],
          zookeepers: ["zookeeper3rd"],
          kafkas: ["kafka3rd", "kafka4th"]    
        }
      },
      baseimage_tag: "x86_64-1.0.0-rc1"
    }


## The method for running just a play, not the entire playbook

The script will create an ansible inventory file named runhosts at the very
first time you run the playbook, the inventory file will be place at a
directory named "run" at the root directory of the playbook. This file will be
updated in later runs if there are changes such as adding or removing hosts.
With this file, if you like to run only few plays, you will be able to do
that by following the example below:

    ansible-playbook -i run/runhosts -e "mode=apply env=bc1st" setupfabric.yml
      --<skip->tags "certsetup"

The above command will use the runhosts inventory file and only run play
named certsetup, all other plays in the play books will be skipped. All
available plays can be found in roles directory, each directory name is
a name can be used in either --tags to be executed or --skip-tags not to
be executed.


## Next Steps

### Check that everything is running correctly

ToDo

## Cleanup

Once you're done with it, don't forget to nuke the whole thing::

    ansible-playbook -e "mode=destroy env=bc1st" setupfabric.yml

The above command will destroy all the fabric resources created such as
the executables on the build machines and all the fabric containers on
all the servers.

If you created the entire environment on your cloud, and you do not
want these machines any more, execute the following command to get rid
of all the servers::

    ansible-playbook -e "mode=destroy env=os password=XXXXX" provcluster.yml
        
## ssh-agent

Since ansible access either the virtual machines that you create on a
cloud or machines that you may already have by using ssh, setting up
ssh-agent on the ansible controller is very important, without doing
this most likely, the script will fail to connect to your servers.
Follow the steps below to setting your ssh-agent on ansible controller
which should be always the machine that you run the ansible script.

1. Create a ssh key pair (only do this once)::

    ssh-keygen -t rsa -f ~/.ssh/fd
        
2. Run the command once in a session in which you run the ansible script

    eval $(ssh-agent -s)
    ssh-add ~/.ssh/fd

3. For the servers created in the cloud, this step is already done for
you. For the existing servers, you will need to make sure that the fd.pub
key is in the file ~/.ssh/authorized_keys. Otherwise, the servers will
reject the ssh connection from ansible controller.


## Cloud environment file vars/os.yml

In vars directory, this project comes with an OpenStack environment file.
This file is used to provision virtual servers on OpenStack cloud. Here
is the file, each field gets explained after the file::

    auth: {
      auth_url: "https://salesdemo-sjc.openstack.blueboxgrid.com:5000/v2.0",
      username: "litong01",
      password: "{{ password }}",
      project_name: "Interop"
    }

    # This variable defines cloud provision attributes
    cluster: {
      target_os: "ubuntu",
      image_name: "Ubuntu 16.04",
      region_name: "",
      ssh_user: "ubuntu",
      availability_zone: "compute_enterprise",
      validate_certs: True,
      private_net_name: "demonet",
      flavor_name: "m1.medium",
      public_key_file: "/home/ubuntu/.ssh/fd.pub",
      private_key_file: "/home/ubuntu/.ssh/fd",
      # This variable indicate what IP should be used, only valid values are
      # private_ip or public_ip
      node_ip: "public_ip",
  
      container_network: {
        Network: "172.17.0.0/16",
        SubnetLen: 24,
        SubnetMin: "172.17.0.0",
        SubnetMax: "172.17.255.0",
        Backend: {
          Type: "udp",
          Port: 8285
        }
      },

      # the section defines preallocated IP addresses for each node, if there is no
      # preallocated IPs, leave it blank
      node_ips: ["169.45.102.186", "169.45.102.187", "169.45.102.188"],

      # fabric network node names expect to be using a clear pattern, this defines
      # the prefix for the node names. 
      name_prefix: "fabric",
      domain: "fabricnet",

      # stack_size determines how many virtual or physical machines we will have
      # each machine will be named ${name_prefix}001 to ${name_prefix}${stack_size} 
      stack_size: 3,

      etcdnodes: ["fabric001", "fabric002", "fabric003"],
      builders: ["fabric001"],
  
      flannel_repo: "https://github.com/coreos/flannel/releases/download/v0.7.1/flannel-v0.7.1-linux-amd64.tar.gz",
      etcd_repo: "https://github.com/coreos/etcd/releases/download/v3.2.0/etcd-v3.2.0-linux-amd64.tar.gz",
    
      go_ver: "1.7.5",
      # If volume want to be used, specify a size in GB, make volume size 0 if wish
      # not to use volume from your cloud
      volume_size: 0,
      # cloud block device name presented on virtual machines.
      block_device_name: "/dev/vdb"
    }

The auth section provides the information about your OpenStack account,
the information here should come from your OpenStack cloud provider.

The cluster section provides information about the servers that you like
to create, also the information how to access them.

    target_os: operating system that your servers will be using
    image_name: cloud image you like to use to create virtual servers.
    ssh_user: user id for ssh log in,
    availability_zone: OpenStack availability zone
    validate_certs: if validate the certificates when access servers.
    private_net_name: private network name where servers being created on
    flavor_name: virtual server specs
    public_key_file: ssh public key file
    private_key_file: ssh private key file
    node_ip: use either private_ip or public_ip when access the servers
    node_ips: preallocated ip addresses for each server
    container_network: overlay network settings, do not change this
      unless you absolutely know what you are doing
    name_prefix: how to name virtual servers, can be any character except dot
    domain: the fabric network domain name, can be any character except dot
    stack_size: how many virtual servers to create,

    etcdnodes: which servers to install etcd services
    builders: which server to be used for building hyperledger fabric
  
    flannel_repo: where to download flanneld
    etcd_repo: where to download etcd
    
    go_ver: version of golang to be installed
    volume_size: future use
    block_device_name: future use
