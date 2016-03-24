# Staring the Demonstration POD Manually
This document describes the steps to manually bring up the
the CORD POD demonstration rack, currently (March 21, 2016)
known as **pod 2**. This assumes a _clean_ start in that
it does not cover multiple failure scenarios nor how to
recover in the event of a failure, where ofter the recovery
can be be wipe the system and start over.

## Power
Supply power to all devices, but do not turn on the devices. Some devices will turn on automatically as soon as they receive power.

## Logging on to POD Servers

## Starting the Head Node
The **Head** node is a compute node in a CORD POD that runs
additionally software including Ubuntu's Metal as a Service (MAAS), OpenStack (OS), Everything as a Service (XOS), and
multiple instances of Open Networking Operating System (ONOS).

#### Power Up
When ready power on the head compute node. This may take several
minutes to boot as these hosts boot slowly. This host should
automatically boot from the disk

#### Restart Metal As A Service (MAAS)
MAAS is a product from Ubuntu that provides Preboot Execution Environment (PXE) boot services, Dynamic Host Configuration Protocol
(DHCP), and Dynamic Name Service (DNS) capabilities. For some reason
yet to be determined MAAS does not automatically start when the
system starts. It thinks it does, but it doesn't. So the MAAS
services need to be manually restarted.

To restart the services login to cord-r2-s1 (head node) and issue
the following commands:

    sudo service maas-clusterd restart
    sudo service maas-regiond restart
    sudo service maas-proxy restart
    sudo service maas-dhcpd restart

As each service starts it will report that it stopped and restarted
the process as in the example below:

    $ sudo service maas-regiond restart
    maas-regiond stop/waiting
    maas-regiond start/running

At this point MAAS should be up and running an ready to provide
services to the rest of the POD.

#### Verify Docker Containers running
Two docker containers are utilized on the head node (cord-r2-s1) to
provide automation around MAAS and DHCP. These docker containers do
start automatically, but it is a good idea to verify they are running
as in the example below: (___Note: currently only a single docker
container is configured to run on cord-r2-s1 as automation of host
deployment has been temporarily disabled.___)

    $ sudo docker ps
    CONTAINER ID        IMAGE                                     COMMAND                  CREATED             STATUS              PORTS                    NAMES
    3d03bf29801e        cord/maas-dhcp-harvester:0.1-prerelease   "python /dhcpharveste"   10 days ago         Up 19 hours         0.0.0.0:8954->8954/tcp   harvester

#### Starting OpenStack
As part of CORD's Everything as a Service (XOS) orchestration
application, OpenStack is used to provision network function
virtualization (NFV). During the original installation of CORD XOS
OpenStack was installed on a series of virtual machines (VM). These
VMs do not automatically restart when the head node restarts.

On the head node (cord-r2-s1) there is a script that can be used to
restart the VMs. This script essentially does a `virsh start` on each
VM and then waits until all the VMs have requested and received IP
address from the DHCP server. This script is named `restart-vms.sh`
and can be found in `/home/ubuntu/ons-demo-files/bin`.

This script can be run multiple times without harm and will report if
all the VMs are not successfully started.

    $ ./ons-demo-files/bin/restart-vms.sh
    Domain ceilometer started
    Domain glance started
    Domain juju started
    Domain keystone started
    Domain mysql started
    Domain nagios started
    Domain neutron-api started
    Domain nova-cloud-controller started
    Domain onos-cord started
    Domain openstack-dashboard started
    Domain rabbitmq-server started
    Domain xos started
    Domain onos-fabric started

    INFO: Waiting for VMs to start
    INFO: Verifing all VMs started
    INFO: Looks like all VM started correctly

#### Boot Remainder of POD hosts
At this point the remainder of the POD hosts can be started. Power
them on and wait until they are fully booted and SSH connectivity
is allowed.

___NOTE: The switches are automatically powered on when powered
is applied to the POD.___

## Starting the Leaf-Spine Fabric
The leaf-spine network fabric provides connectivity between the
compute hosts 40G network interfaces. This connectivity traverses
the OpenFlow (OF) switches in the POD rack.

#### Compute Node (cord-r2-s2) Bridge
Before we can start the fabric some other CORD services need to be
started as they create / manage an OVS bridge (`br-int`) on
cord-r2-s2. When this bridge is created it adds cord-r2-s2's
eth0 (40G) interface to the bridge and thus this needs to be
configured before moving forward on the fabric.

The bridge is controlled by an instance of ONOS running in a VM on
the head (cord-r2-s1) node. To start this controller log in to
cord-r2-s1 and and then to the VM onos-cord. Once on the VM,
used`docker-compose` to start the ONOS instance. The example below
assumes you are already logged in to cord-r2-s1.

    ssh onos-cord
    cd cord
    docker-compose up -d

The creation and control of this bridge can take some time, so you may have to
wait several minutes. To verify the bridge is operational you can issue the
following command on compute node (cord-r2-s2):

    $ sudo ovs-vsctl show
    c2599bab-38c1-4bec-addc-92235a1f202a
    Bridge br-int
       Controller "tcp:172.18.100.6:6653"
           is_connected: true
       fail_mode: secure
       Port "tap0b5b242f-69"
           Interface "tap0b5b242f-69"
       Port "tap2543002d-e4"
           Interface "tap2543002d-e4"
       Port br-int
           Interface br-int
       Port "eth0"
           Interface "eth0"
       Port vxlan
           Interface vxlan
               type: vxlan
               options: {key=flow, remote_ip=flow}
    ovs_version: "2.3.2"

The critical information retuned from this command is the information about
connectivity to the controller:

    Controller "tcp:172.18.100.6:6653"
        is_connected: true

and the information about the included interface:

    Port "eth0"
        Interface "eth0"

As a consequence of this bridge being established and `eth0` being added to the
bridge an important route is removed by the system. This route enables traffic
to flow from cord-r2-s2, connected to leaf switch 1, to the compute nodes
connected to leaf switch 2. To add this route back to cord-r2-s2 issue the
following command on cord-r2-s2:

    sudo route add -net 10.3.2.0/24 gw 10.3.1.254

#### Starting the Fabric Controller
The leaf-spine fabric is controlled by a custom instance of ONOS. This instance
is **custom** because is contains modified Java source code to support the
multiple cast flows. The source for the custom version can be found in
`/home/ubuntu/ons-demo-files/src/onos.multicast.multichan` and the build
artifacts can be found in `/home/ubuntu/ons-demo-files/onos-1.6.0.ubuntu`, both
on cord-r2-s1.

###### Resetting the ONOS Fabric Configuration
The configuration and some state for ONOS is persisted over restarts, so it is
often the case that the configuration needs to be reset in order to get a clean
start of the fabric so that it is functional. To do this, the ONOS instance
is started and the existing configuration is deleted and a new configuration is
posted. After this the ONOS instance must be restarted as the configuration is
only read and processed by the segment routing application when the application
is started, i.e. when ONOS is started. ___NOTE: technically you might be able
to accomplish this by deactivating and reactivating the segment routing
applications, but restarting ONOS is cleaner.___

Scripts to help support the configuration operations can be found in
`/home/ubuntu/ons-demo-files/bin` on cord-r2-s1. These scripts access the
ONOS REST API via the `curl` executable.

To reset the configuration start ONOS is one terminal logged in to cord-r2-s1
and issue the commands to modify the configuration from another terminal also
logged in to cord-r2-s1.

To start the ONOS instance:

    cd ./ons-demo-files/onos-1.6.0.ubuntu
    ./bin/onos-service

To delete the configuration issue the following command. If there is output
from the command such as a message about not being able to parse a JSON object
that just means that ONOS is not fully initialized yet. Keep issuing the
delete command until there is no output.

    ./ons-demo-files/bin/onos-cfg-delete

Once the delete command succeeds you can verify that the configuration has
been deleted using the get command and you should see the following output:

    $ ./ons-demo-files/bin/onos-cfg-get
    {
        "apps": {},
        "devices": {},
        "hosts": {},
        "links": {},
        "ports": {}
    }

Once the configuration has been deleted you can post the configuration file to
onos as follows:

    ./ons-demo-files/bin/onos-cfg-post /home/ubuntu/ons-demo-files/configs/onos/pod-2/network-config-with-vsg.json

You can verify that the configuration was accepted by issuing an additional get
command, but it is not necessary. After the configuration is posted to ONOS You
can shutdown the ONOS instance either by typing Control-D (^D) on the ONOS CLI
prompt of using the `shutdown` command at the ONOS CLI prompt.

###### Resetting the OF switches
Besides resetting ONOS the OF switches need to be reset as well. This includes
restarting the ofdpa agent and clearing the ASICs. To do that the following
needs to be executed on each switch. Scripts have been installed on the switches
to aid in this process, you can view the scripts to see what they actually do.

If a command *locks* or *freezes* on the switch, as `purge` sometimes does, you
will need to break out of that command using Control-C (^C) and restart the
ofdpa service
using:

    service ofdpa restart

and then repeat the command that *locked* or *froze*.

Log into each of the switches and issue the following commands:

    ./killit
    ./purge

The `killit` command stops any running ofdpa agent and the `purge` command
purges the ASICs.

It is sometimes helpful to keep a terminal open with a session connected to
each switch.

###### Bringing up the Fabric
At this point the components that participate in the fabric have been reset
and are in the state to be restarted. The sequence to start the fabric is
ONOS, spines, and then leaves.

When starting processes for the fabric they are started as background processes
with the `nohup` command so that the processes continue to run in the background
even it the terminal session in which the process was started is terminated. If
you want to see the output of the command you can use the command `tail` on the
file `nohup.out` that is created in the directory from which the process was
started.

###### Starting ONOS
From cord-r2-s1 you can restart ONOS using the following commands:

    cd /home/ubuntu/ons-demo-files/onos-1.6.0.ubuntu
    nohup ./bin/onos-service server &

After ONOS is started you can access the ONOS CLI either by SSH-ing to ONOS:

    ssh -p 8101 karaf@127.0.0.1

with the password `karaf` or by executing the `onos` command from the
directory `/home/ubuntu/ons-demo-files/onos-1.6.0.ubuntu`:

    ./bin/onos

To view the log of ONOS you can use the `log:tail` command at the ONOS CLI. You
can set various logging details using the `log:set` command. See the ONOS
documentation for more details.

    log:set debug org.onosproject.segmentrouting
    log:tail

###### Connecting switches to controller
To connect the switches to the controller you need to issue the `connect`
command on each switch. The order is `spine-1`, `spine-2`, `leaf-1`, `leaf-2`.
In theory the order shouldn't matter, in practice this order seems to provide
the most consistent results.

When starting the switches if an error occurs in the switch log or an error
occurs in the ONOS log this typically means that the fabric will not function
and you have to reset ONOS and the switches, ie. start the process of bringing
up the fabric again.

To connect the switches to the controller issue the following command on each
of the switches. The second command shown, `tail`, is used to view the output
of the command. This is optional and you don't need to issue that command
unless you want to view the output. ___NOTE: it is recommended that you maintain
a terminal session to each switch and view the output so that you can verify
the fabric started without errors.___

    nohup ./connect &
    tail -f nohup.out

###### Fabric UP
At this point the leaf-spine fabric should be up and operational. You can test
that the fabric is working in several ways, including attempting to ping the
fabric gateways from each compute node:

    ping -I eth0 10.3.1.254
    ping -I eth0 10.3.2.254

And by pinging every compute node from every other compute node. There is a
utility script found in `/home/ubuntu/ping-test.sh` on each compute node. This
script will ping each other server as well as some additional servers. You
can execute this once or use the linux `watch` command to repeatedly execute
the script.

    $ ./ping-test.sh
    FROM: 10.3.1.1
        10.3.1.1:       0.060
        10.3.1.2:       0.311
        10.3.2.1:       0.139
        10.3.2.2:       0.140
        10.3.1.254:     2.119
        10.3.2.254:     1.675
        192.168.10.1:
        8.8.8.8:

___NOTE: This ping test pings using the 40G interface, this means on cord-r2-s1
it is expected that `192.168.10.1` and `8.8.8.8` can't be reached. These should
be reachable from the other nodes, but are not currently. This is an outstanding
issue that needs to be resolved.___

Now that the fabric is operational the multicast video should work as it does
not currently depend on CORD.

#### Restarting CORD / XOS
CORD / XOS leverages OpenStack as well as has its own processes. Additionally
CORD / XOS leverages VMs and machines on cord-r2-s2 as well. To bring up CORD /
XOS after a reboot the first step is to restart the virtual service gateway
that was installed on cord-r2-s2. To do this, ssh to cord-r2-s2 and issue the
following command:

    virsh start instance-00000001

The next step is to restart the XOS, VTN, and CORD processes that are running
on VMs off of cord-r2-s1. To do this log in to cord-r2-s1 and issue the
following commands:

    ssh xos
    cd xos/xos/configurations/cord-pod
    make
    make vtn
    make cord

___NOTE: It is recommended that you wait a few minute between each `make`
command to give the solution some time to "settle".___

After XOS has been started, it likely is the case that some OpenStack nova
services need to be restarted as well. You can determine this by using the
`nova list --all-tenants` command on cord-r2-s1. Before issuing the command
be sure to load the nova authentication information into your environment by
sourcing the file `/home/ubuntu/admin-openrc.sh`:

    $ . admin-openrc.sh
    $ nova list --all-tenants
    +--------------------------------------+--------------+---------+------------+-------------+---------------------------------------------------+
    | ID                                   | Name         | Status  | Task State | Power State | Networks                                          |
    +--------------------------------------+--------------+---------+------------+-------------+---------------------------------------------------+
    | 9e1fe7c4-bb83-4b43-ac9f-74a4e3351f19 | mysite_vsg-1 | SHUTOFF | -          | Shutdown    | management=172.27.0.2; mysite_vsg-access=10.0.2.2 |
    +--------------------------------------+--------------+---------+------------+-------------+---------------------------------------------------+

If the service is indicating that it is `SHUTOFF` then the service should be restarted using the `nova start` command.

    nova start 9e1fe7c4-bb83-4b43-ac9f-74a4e3351f19

# Things of Note
At this point CORD should be up and running. In practice there is still an
issue with pinging from the vSG to the internet. I think this is an issue with
routes and or IP tables that somehow doesn't get reset when the system reboots.
I have not debugged this fully and it really needs to be addressed.

## Trouble shooting
Below are some useful command to help troubleshoot when things go bad. I have
included examples of expected output to which you can compare your results.
How to resolve any issue depends on the issues, so good luck.

on cord-r2-s1:

    $ nova net-list
    +--------------------------------------+-------------------+------+
    | ID                                   | Label             | CIDR |
    +--------------------------------------+-------------------+------+
    | b9f726b1-a399-4d01-b4af-1908322e2431 | management        | None |
    | cde5fe39-f2ea-4daa-ac1f-6ea92f574f89 | mysite_vsg-access | None |
    +--------------------------------------+-------------------+------+

on cord-r2-s1:

    $ nova list --all-tenants
    +--------------------------------------+--------------+--------+------------+-------------+---------------------------------------------------+
    | ID                                   | Name         | Status | Task State | Power State | Networks                                          |
    +--------------------------------------+--------------+--------+------------+-------------+---------------------------------------------------+
    | 9e1fe7c4-bb83-4b43-ac9f-74a4e3351f19 | mysite_vsg-1 | ACTIVE | -          | Running     | management=172.27.0.2; mysite_vsg-access=10.0.2.2 |
    +--------------------------------------+--------------+--------+------------+-------------+---------------------------------------------------+


on cord-r2-s1: (use `karaf` as password)

    $ ssh -p 8101 karaf@onos-cord
    Password authentication
    Password:
    Welcome to Open Network Operating System (ONOS)!
     ____  _  ______  ____
    / __ \/ |/ / __ \/ __/
    / /_/ /    / /_/ /\ \
    \____/_/|_/\____/___/

    Documentation: wiki.onosproject.org
    Tutorials:     tutorials.onosproject.org
    Mailing lists: lists.onosproject.org

    Come help out! Find out how at: contribute.onosproject.org

    Hit '<tab>' for a list of available commands
    and '[cmd] --help' for help on a specific command.
    Hit '<ctrl-d>' or type 'system:shutdown' or 'logout' to shutdown ONOS.

    onos> cordvtn-
    cordvtn-flush-rules    cordvtn-node-check     cordvtn-node-delete    cordvtn-node-init      cordvtn-nodes
    onos> cordvtn-node
    cordvtn-node-check     cordvtn-node-delete    cordvtn-node-init      cordvtn-nodes
    onos> cordvtn-nodes
    hostname=cord-r2-s2.cord.lab, hostMgmtIp=10.2.0.2/24, dpIp=10.3.1.2/24, br-int=of:0000000000000001, dpIntf=eth0, init=COMPLETE
    Total 1 nodes

on cord-r2-s1:

    $ nova service-list
    +------------------+-----------------------+----------+---------+-------+----------------------------+-----------------+
    | Binary           | Host                  | Zone     | Status  | State | Updated_at                 | Disabled Reason |
    +------------------+-----------------------+----------+---------+-------+----------------------------+-----------------+
    | nova-consoleauth | nova-cloud-controller | internal | enabled | up    | 2016-03-24T21:16:47.000000 | -               |
    | nova-scheduler   | nova-cloud-controller | internal | enabled | up    | 2016-03-24T21:16:47.000000 | -               |
    | nova-cert        | nova-cloud-controller | internal | enabled | up    | 2016-03-24T21:16:37.000000 | -               |
    | nova-conductor   | nova-cloud-controller | internal | enabled | up    | 2016-03-24T21:16:47.000000 | -               |
    | nova-compute     | cord-r2-s2            | nova     | enabled | up    | 2016-03-24T21:16:46.000000 | -               |
    +------------------+-----------------------+----------+---------+-------+----------------------------+-----------------+

on cord-r2-s1:

    $ juju status --format=tabular
    [Services]
    NAME                  STATUS  EXPOSED CHARM
    ceilometer            error   false   cs:trusty/ceilometer-17
    ceilometer-agent              false   cs:trusty/ceilometer-agent-13
    glance                error   false   cs:trusty/glance-28
    juju-gui              unknown true    cs:trusty/juju-gui-52
    keystone              error   false   cs:trusty/keystone-33
    mongodb               unknown false   cs:trusty/mongodb-33
    mysql                 error   false   cs:trusty/percona-cluster-31
    nagios                unknown false   cs:trusty/nagios-10
    neutron-api           error   false   cs:~cordteam/trusty/neutron-api-1
    nova-cloud-controller error   false   cs:trusty/nova-cloud-controller-64
    nova-compute          active  false   cs:~cordteam/trusty/nova-compute-2
    nrpe                          false   cs:trusty/nrpe-4
    ntp                           false   cs:trusty/ntp-14
    openstack-dashboard   active  false   cs:trusty/openstack-dashboard-19
    rabbitmq-server       error   false   cs:trusty/rabbitmq-server-42

    [Units]
    ID                      WORKLOAD-STATE AGENT-STATE VERSION MACHINE PORTS                                   PUBLIC-ADDRESS        MESSAGE
    ceilometer/0            error          idle        1.25.3  7       8777/tcp                                ceilometer            hook failed: "config-changed"
      nrpe/3                unknown        idle        1.25.3                                                  ceilometer
    glance/0                error          idle        1.25.3  4       9292/tcp                                glance                hook failed: "config-changed"
      nrpe/5                unknown        idle        1.25.3                                                  glance
    juju-gui/0              unknown        idle        1.25.3  0       80/tcp,443/tcp                          juju
    keystone/0              error          idle        1.25.3  3                                               keystone              hook failed: "config-changed"
      nrpe/4                unknown        idle        1.25.3                                                  keystone
    mongodb/0               unknown        idle        1.25.3  7       27017/tcp,27019/tcp,27021/tcp,28017/tcp ceilometer
    mysql/0                 error          idle        1.25.3  1                                               mysql                 hook failed: "config-changed"
      nrpe/6                unknown        idle        1.25.3                                                  mysql
    nagios/0                unknown        idle        1.25.3  8       80/tcp                                  nagios
    neutron-api/0           error          idle        1.25.3  9       9696/tcp                                neutron-api           hook failed: "config-changed"
      nrpe/2                unknown        idle        1.25.3                                                  neutron-api
    nova-cloud-controller/0 error          idle        1.25.3  5       3333/tcp,8773/tcp,8774/tcp,9696/tcp     nova-cloud-controller hook failed: "config-changed"
      nrpe/8                unknown        idle        1.25.3                                                  nova-cloud-controller
    nova-compute/0          active         idle        1.25.3  10                                              10.2.0.2              Unit is ready
      ceilometer-agent/0    waiting        idle        1.25.3                                                  10.2.0.2              Incomplete relations: ceilometer
      nrpe/7                unknown        idle        1.25.3                                                  10.2.0.2
      ntp/0                 unknown        idle        1.25.3                                                  10.2.0.2
    openstack-dashboard/0   active         idle        1.25.3  6       80/tcp,443/tcp                          openstack-dashboard   Unit is ready
      nrpe/0                unknown        idle        1.25.3                                                  openstack-dashboard
    rabbitmq-server/0       error          idle        1.25.3  2       5672/tcp                                rabbitmq-server       hook failed: "config-changed"
      nrpe/1                unknown        idle        1.25.3                                                  rabbitmq-server

    [Machines]
    ID         STATE   VERSION DNS                   INS-ID                       SERIES HARDWARE
    0          started 1.25.3  juju                  manual:                      trusty arch=amd64 cpu-cores=1 mem=2001M
    1          started 1.25.3  mysql                 manual:mysql                 trusty arch=amd64 cpu-cores=2 mem=3953M
    2          started 1.25.3  rabbitmq-server       manual:rabbitmq-server       trusty arch=amd64 cpu-cores=2 mem=3953M
    3          started 1.25.3  keystone              manual:keystone              trusty arch=amd64 cpu-cores=2 mem=3953M
    4          started 1.25.3  glance                manual:glance                trusty arch=amd64 cpu-cores=2 mem=3953M
    5          started 1.25.3  nova-cloud-controller manual:nova-cloud-controller trusty arch=amd64 cpu-cores=2 mem=3953M
    6          started 1.25.3  openstack-dashboard   manual:openstack-dashboard   trusty arch=amd64 cpu-cores=1 mem=2001M
    7          started 1.25.3  ceilometer            manual:ceilometer            trusty arch=amd64 cpu-cores=1 mem=2001M
    8          started 1.25.3  nagios                manual:nagios                trusty arch=amd64 cpu-cores=1 mem=2001M
    9          started 1.25.3  neutron-api           manual:neutron-api           trusty arch=amd64 cpu-cores=2 mem=3953M
    10         started 1.25.3  10.2.0.2              manual:10.2.0.2              trusty arch=amd64 cpu-cores=24 mem=128829M
