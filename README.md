# mbx 🐒📦

<img src="https://raw.githubusercontent.com/shapeblue/mbx/main/doc/images/box-start.png" style="width:500px;">

MonkeyBox `mbx` enables building CloudStack packages and deploying CloudStack
dev and qa environment using pre-built DHCP-enabled VM templates.

Table of Contents
=================

* [Architecture](#architecture)
    * [Storage](#storage)
    * [Networking](#networking)
    * [Deployment](#deployment)
* [Installation and Setup](#installation-and-setup)
    * [Setup NFS Storage](#setup-nfs-storage)
    * [Setup KVM](#setup-kvm)
    * [Setup mbx](#setup-mbx)
    * [Create Storage Gold-Masters](#create-storage-gold-masters)
* [Using mbx](#using-mbx)
* [CloudStack Development](#cloudstack-development)
    * [Install Development Tools](#install-development-tools)
    * [Setup MySQL Server](#setup-mysql-server)
    * [Setup NFS storage](#setup-nfs-storage-1)
    * [Dev: Build and Test CloudStack](#dev-build-and-test-cloudstack)
    * [Debugging CloudStack](#debugging-cloudstack)
* [Contributing](#contributing)
* [Troubleshooting](#troubleshooting)
    * [iptables](#iptables)

## Architecture

![mbx architecture](doc/images/arch.png)

A `mbx` environment consists of VMs that runs the CloudStack management server
and hypervisor hosts. These VMs are provisioned on a local host-only `monkeynet`
network which is a /16 nat-ed RFC1918 IPv4 network. The diagram above shows how
nested guest VMs and virtual router are plugged in nested-virtual networks that
run in a nested KVM host VM.

### Storage

`mbx` requires NFS storage to be setup and exported for the base path
`/export/testing` for environment-specific primary and secondary storages.

A typical `mbx` environment deployment makes copy of a CloudStack
version-specific gold-master directory that generally containing two empty primary
storage directory (`primary1` and `primary2`) and one secondary storage
directory (`secondary`). The secondary storage directory must be seeded
with CloudStack version-specific `systemvmtemplates`. The `systemvmtemplate` is
then used to create system VMs such as the Secondary-Storage VM, Console-Proxy
VM and Virtual Router in an `mbx` environment.

### Networking

`mbx` requires a local 172.20.0.0/16 natted network such that the VMs on this
network are only accessible from the workstation/host but not by the outside
network. The `mbx init` command initialises this network.

    External Network
      .                     +-----------------+
      |              virbr1 | MonkeyBox VM1   |
      |                  +--| IP: 172.20.0.10 |
    +-----------------+  |  +-----------------+
    | Host x.x.x.x    |--+
    | IP: 172.20.0.1  |  |  +-----------------+
    +-----------------+  +--| MonkeyBox VM2   |
                            | IP: 172.20.x.y  |
                            +-----------------+

The 172.20.0.0/16 RFC1918 private network is used as the other 192.168.x.x and
10.x.x.x CIDRs may be already in by VPN, lab resources and office/home networks.

To keep the setup simple all MonkeyBox VMs have a single nic which can be
used as a single physical network in CloudStack that has the public, private,
management/control and storage networks. A complex setup is possible by adding
multiple virtual networks and nics on them.

### Deployment

For QA env, `mbx` will deploy a single `mgmt` VM that runs the management
server, the usage server, MySQL server, marvin integration tests etc. and two
hypervisor host VMs.

For Dev env, `mbx` will deploy a single hypervisor host VM and the management
server, usage server, MySQL server etc. are all run from the workstation/host by
the developer.

For both QA and Dev environments, the environment-specific NFS storage are
generally directories under `/export/testing` which serve as both primary and
secondary storage.

The `mbx` templates are initialised and downloaded at
`/export/monkeybox/templates/`.

The `mbx` environments, their configurations and VM disks are hosted at
`/export/monkeybox/boxes/`.

## Installation and Setup

`mbx` has been tested against Ubuntu 20.04 LTS with KVM+QEMU 4.2 and NFS storage.

We recommend at least 32GB RAM with x86_64 Intel VT-x or AMD-V enabled CPU on the
workstation/host where `mbx` is used and uninstall any other hypervisors such as
VirtualBox or VMware workstation.

Additional notes:
- Default password for all the root user is `P@ssword123`.
- `mbx` requires docker for building packages: https://docs.docker.com/engine/install/ubuntu/

### Setup NFS Storage

    apt-get install nfs-kernel-server quota sshpass wget jq
    echo "/export  *(rw,async,no_root_squash,no_subtree_check)" > /etc/exports
    mkdir -p /export/testing
    exportfs -a
    sed -i -e 's/^RPCMOUNTDOPTS="--manage-gids"$/RPCMOUNTDOPTS="-p 892 --manage-gids"/g' /etc/default/nfs-kernel-server
    sed -i -e 's/^STATDOPTS=$/STATDOPTS="--port 662 --outgoing-port 2020"/g' /etc/default/nfs-common
    echo "NEED_STATD=yes" >> /etc/default/nfs-common
    sed -i -e 's/^RPCRQUOTADOPTS=$/RPCRQUOTADOPTS="-p 875"/g' /etc/default/quota
    service nfs-kernel-server restart

### Setup KVM

    apt-get install qemu-kvm libvirt-daemon bridge-utils cpu-checker nfs-kernel-server quota
    kvm-ok

Fixing permissions for libvirt-qemu:

    sudo getfacl -e /export
    sudo setfacl -m u:libvirt-qemu:rx /export

Install Libvirt NSS for name resolution:

    apt-get install libnss-libvirt

Next, add the following so that `grep -w 'hosts:' /etc/nsswitch.conf` returns:

    files libvirt libvirt_guest dns mymachines

Install `virt-manager`, the virtual machine manager graphical tool to manage VMs
on your machine:

    apt-get install virt-manager

![VM Manager](doc/images/virt-manager.png)

### Setup `mbx`

    git clone https://github.com/shapeblue/mbx /export/monkeybox

    # Enable mbx under $PATH, for bash:
    echo export PATH="/export/monkeybox:$PATH" >> ~/.bashrc
    # Enable mbx under $PATH, for zsh:
    echo export PATH="/export/monkeybox:$PATH" >> ~/.zshrc

    # Initialise `mbx` by opening in another shell:
    mbx init

The `mbx init` is idempotent and can be used to update templates and domain xml
definitions.

The `mbx init` command initialises this network. You can check and confirm the
network using:

    $ virsh net-list
    Name                 State      Autostart     Persistent
    ----------------------------------------------------------
    default              active     yes           yes
    monkeynet            active     yes           yes

Alternatively, you may open `virt-viewer` manager and click on:

    Edit -> Connection Details -> Virtual Networks

You may also manually add/configure a virtual network with NAT in 172.20.0.0/16
like below:

![VM Manager Virt Network](doc/images/virt-net.png)

This will create a virtual network with NAT and CIDR 172.20.0.0/16, the gateway
`172.20.0.1` is also workstation/host's virtual bridge IP. The virtual network's
bridge name `virbrX`may be different and it does not matter as long as you've a
NAT-enabled virtual network in 172.20.0.0/16.

    Your workstation/host IP address is `172.20.0.1`.

### Create Storage Gold-Masters

After setting up NFS on the workstation host, you need to create a
CloudStack-version specific storage golden master directory that contains two
primary storages and secondary storage folder with the systemvmtemplate for a
specific version of CloudStack seeded. The storage golden master is used as
storage source of a mbx environment during `mbx deploy` command execution.

Note: This is required one-time only for a specific version of CloudStack.

For example, the following is needed only one-time for creating a golden master
storage directory for 4.15 version:

    mkdir -p /export/testing
    # Create directory layout for a specific ACS version under /export/testing
    mkdir -p /export/testing/4.15/{primary1,primary2,secondary}
    # Get the systemvm templates
    cd /export/testing/4.15
    wget http://packages.shapeblue.com/systemvmtemplate/4.15/systemvmtemplate-4.15.1-kvm.qcow2.bz2
    wget http://packages.shapeblue.com/systemvmtemplate/4.15/systemvmtemplate-4.15.1-vmware.ova
    wget http://packages.shapeblue.com/systemvmtemplate/4.15/systemvmtemplate-4.15.1-xen.vhd.bz2
    wget http://packages.shapeblue.com/systemvmtemplate/4.15/md5sum.txt
    # Check the downloaded templates, it should say OK for the three templates
    md5sum --check md5sum.txt
    # Seed template in the secondary folder for 4.15
    /export/monkeybox/files/setup-systemvmtemplate.sh -m /export/testing/4.15/secondary -f systemvmtemplate-4.15.1-kvm.qcow2.bz2 -h kvm
    /export/monkeybox/files/setup-systemvmtemplate.sh -m /export/testing/4.15/secondary -f systemvmtemplate-4.15.1-vmware.ova -h vmware
    /export/monkeybox/files/setup-systemvmtemplate.sh -m /export/testing/4.15/secondary -f systemvmtemplate-4.15.1-xen.vhd.bz2 -h xenserver
    # Cleanup downloaded files
    rm -fv md5sum.txt systemvmtemplate*

## Using `mbx`

`mbx` tool can be used to build CloudStack packages, deploy dev or QA
environments with KVM, VMware, XenServer and XCP-ng hypervisors, and run
smoketests on them.

    $ mbx
    MonkeyBox 🐵 v0.1
    Available commands are:
      init: initialises monkeynet and mbx templates
      package: builds packages from a git repo and sha/tag/branch
      list: lists available environments
      deploy: deploys QA env with two monkeybox VMs, configures storage, creates marvin cfg file
      launch: launches QA env zone using environment's marvin cfg file
      test: start marvin tests
      dev: deploys dev env with a single monkeybox VM, configures storage, creates marvin cfg file
      agentscp: updates KVM agent in dev environment using scp and restarts it
      ssh: ssh into a mbx VM
      stop: stop all env VMs
      start: start all env VMs
      destroy: destroy environment

0. On first run, initialise networking and templates, run:

    mbx init

1. To list available environments and `mbx` templates (mbxts) run:

    mbx list

2. To deploy an environment run:

    mbx deploy <name of env, default: mbxe> <mgmt server template, default: mbxt-kvm-centos7> <hypervisor template, default: mbxt-kvm-centos7> <repo, default: http://packages.shapeblue.com/cloudstack/upstream/centos7/4.15> <storage source, default: /export/testing/4.15>

Example to deploy test matrix (kvm, vmware, xenserver) environments:

    mbx deploy 415-kenv mbxt-kvm-centos7 mbxt-kvm-centos7 # deploys 4.15 + KVM CentOS7 env
    mbx deploy 415-venv mbxt-kvm-centos7 mbxt-vmware67u3  # deploys 4.15 + VMware67u3 env
    mbx deploy 415-xenv mbxt-kvm-centos7 mbxt-xenserver71 # deploys 4.15 + XenServer71 env

More examples with specific repositories and custom storage source: (custom storage source must exist)

    mbx deploy 416-snapshot mbxt-kvm-centos7 mbxt-kvm-centos7 http://download.cloudstack.org/testing/nightly/latest/centos7/4.16 /export/testing/4.16.0

3. Once `mbx` environment is deployed, to launch a zone run:

    mbx launch <name of the env, run `mbx list` for env name>

4. To run smoketests, run:

    mbx list # find your environment
    mbx ssh <name of the mbx VM>
    cd /marvin # here you'll find smoketests.sh to run smoketests
    bash -x smoketests.sh

5. To destroy your mbx environment, run:

    mbx destroy <name of the env, see mbx list for env name>

## CloudStack Development

This section cover how a developer can run management server and MySQL server
locally to do local CloudStack development along side an IDE.

For developer env, it is recommended that you run your favourite IDE such as
IntelliJ IDEA, text-editors, your management server, MySQL server and NFS server
(secondary and primary storages) on your workstation (not in a VM) where these
services can be accessible to VMs, KVM hosts etc. at your host IP `172.20.0.1`.

To ssh into deployed VMs (with NSS configured), you can login simply using:

    $ mbx ssh <name of VM>

Refer to hackerbook for up-to-date guidance on CloudStack development:
https://github.com/shapeblue/hackerbook

### Install Development Tools

Run this:

    $ sudo apt-get install openjdk-11-jdk maven python-mysql.connector libmysql-java mysql-server mysql-client bzip2 nfs-common uuid-runtime python-setuptools ipmitool genisoimage

Setup IntelliJ (recommended) or any IDE of your choice. Get IntelliJ IDEA
community edition from:

    https://www.jetbrains.com/idea/download/#section=linux

Install pyenv, jenv as well.

Setup `aliasrc` that defines some useful bash aliases, exports and utilities
such as `agentscp`. Run the following while in the directory root:

    $ echo "source $PWD/aliasrc" >> ~/.bashrc
    $ echo "source $PWD/aliasrc" >> ~/.zshrc

You may need to `source` your shell's rc/profile or relaunch shell/terminal
to use `agentscp`.

### Setup MySQL Server

After installing MySQL server, configure the following settings in its config
file such as at `/etc/mysql/mysql.conf.d/mysqld.cnf` and restart mysql-server:

    [mysqld]

    sql-mode="STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION,ERROR_FOR_DIVISION_BY_ZERO,NO_ZERO_DATE,NO_ZERO_IN_DATE,NO_ENGINE_SUBSTITUTION"
    server_id = 1
    innodb_rollback_on_timeout=1
    innodb_lock_wait_timeout=600
    max_connections=1000
    log-bin=mysql-bin
    binlog-format = 'ROW'

### Setup NFS storage

After installing nfs server, configure the exports:

    echo "/export  *(rw,async,no_root_squash,no_subtree_check)" > /etc/exports
    mkdir -p /export/testing/primary /export/testing/secondary

Beware: For Dev env, before deploying a zone on your monkeybox environment, make
sure to seed the correct systemvmtemplate applicable for your branch. In your
cloned CloudStack git repository you can use the `cloud-install-sys-tmplt` to
seed the systemvmtemplate.

The following is an example to setup `4.15` systemvmtemplate which you should
run after deploying CloudStack db: (please use CloudStack branch/version specific
systemvmtemplate)

    cd /path/to/cloudstack/git/repo
    wget http://packages.shapeblue.com/systemvmtemplate/4.15/systemvmtemplate-4.15.1-kvm.qcow2.bz2
    ./scripts/storage/secondary/cloud-install-sys-tmplt \
          -m /export/testing/secondary -f systemvmtemplate-4.15.1-kvm.qcow2.bz2 \
          -h kvm -o localhost -r cloud -d cloud

### Dev: Build and Test CloudStack

It's assumed that the directory structure is something like:

        /
        ├── $home/lab/cloudstack
        └── /export/monkeybox

Fork the repository at: github.com/apache/cloudstack, or get the code:

    $ git clone https://github.com/apache/cloudstack.git

Noredist CloudStack builds requires additional jars that may be installed from:

    https://github.com/shapeblue/cloudstack-nonoss

Clone the above repository and run the install.sh script, you'll need to do
this only once or whenver the noredist jar dependencies are updated in above
repository.

Build using:

    $ mvn clean install -Dnoredist -P developer,systemvm

Deploy database using:

    $ mvn -q -Pdeveloper -pl developer -Ddeploydb

Run management server using:

    $ mvn -pl :cloud-client-ui jetty:run  -Dnoredist -Djava.net.preferIPv4Stack=true

Install marvin:

    $ sudo pip install --upgrade tools/marvin/dist/Marvin*.tar.gz

While in CloudStack's repo's root/top directory, run the folllowing to copy
agent scripts, jars, configs to your KVM host:

    $ cd /path/to/git-repo/root
    $ mbx agentscp 172.20.1.10  # Use the appropriate KVM box IP

Deploy datacenter using:

    $ python tools/marvin/marvin/deployDataCenter.py -i ../monkeybox/adv-kvm.cfg

Example, to run a marvin test:

    $ nosetests --with-xunit --xunit-file=results.xml --with-marvin --marvin-config=../monkeybox/adv-kvm.cfg -s -a tags=advanced --zone=KVM-advzone1 --hypervisor=KVM test/integration/smoke/test_vm_life_cycle.py

Note: Use nosetests-2.7 to run a smoketest, if you've nose installed for both Python2.7 and Python3.x in your environment.

When you fix an issue, rebuild cloudstack and push new changes to your KVM host
using `agentscp` which will also restart the agent:

    $ agentscp 172.20.1.10

Using IDEA IDE:
- Import the `cloudstack` directory and select `Maven` as build system
- Go through the defaults, in the profiles page at least select noredist, vmware
  etc.
- Once IDEA builds the codebase cache you're good to go!

### Debugging CloudStack

Prior to starting CloudStack management server using mvn (or otherwise), export
this on your shell:

    export MAVEN_OPTS="$MAVEN_OPTS -Xdebug -Xrunjdwp:transport=dt_socket,address=8787,server=y,suspend=n"

To remote-debug the KVM agent, put the following in
`/etc/default/cloudstack-agent` in your monkeybox and restart cloudstack-agent:

    JAVA=/usr/bin/java -Xdebug -Xrunjdwp:transport=dt_socket,address=8787,server=y,suspend=n

The above will ensure that JVM with start with debugging enabled on port 8787.
In IntelliJ, or your IDE/editor you can attach a remote debugger to this
address:port and put breakpoints (and watches) as applicable.

## Contributing

Report issues on https://github.com/shapeblue/mbx/issues

Send a pull request on https://github.com/shapeblue/mbx

## Troubleshooting

### iptables

Should your datacenter deployment fail due to the KVM host unable to reach your management server, it might be due to iptable rules.

If you see this in your hosts agent.log:

    java.net.NoRouteToHostException: No route to host
            at sun.nio.ch.Net.connect0(Native Method)
            at sun.nio.ch.Net.connect(Net.java:454)
            at sun.nio.ch.Net.connect(Net.java:446)
            at sun.nio.ch.SocketChannelImpl.connect(SocketChannelImpl.java:648)
            at com.cloud.utils.nio.NioClient.init(NioClient.java:56)
            at com.cloud.utils.nio.NioConnection.start(NioConnection.java:95)
            at com.cloud.agent.Agent.start(Agent.java:263)
            at com.cloud.agent.AgentShell.launchAgent(AgentShell.java:410)
            at com.cloud.agent.AgentShell.launchAgentFromClassInfo(AgentShell.java:378)
            at com.cloud.agent.AgentShell.launchAgent(AgentShell.java:362)
            at com.cloud.agent.AgentShell.start(AgentShell.java:467)
            at com.cloud.agent.AgentShell.main(AgentShell.java:502)

And a telnet from host to management server on port gives this result:

    $ telnet 172.20.0.1 8250
    Trying 172.20.0.1...
    telnet: connect to address 172.20.0.1: No route to host

Clearing your iptables and setting new rules should take care of the issue. (Tested on Ubuntu 17.10)

Run the following commands as su or with sudo powers.

Backup and then flush your rules and delete any user-defined chains:

    $ iptables -t nat -F && iptables -t nat -X
    $ iptables -t filter -F && iptables -t filter -X

Add new rules by running the two scripts located in docs/scripts to set up new nat and filter rules,
ensuring that the network name (virbr1) in filter.table matches your management server IP:

    $ bash -x <script>

Alternatively, add each rule separately.

Finally, save your iptables.

Ubuntu:

    iptables-save

and if using iptables-persistent:

    service iptables-persistent save

CentOS 6 and older (CentOS 7 uses FirewallD by default):

    service iptables save
