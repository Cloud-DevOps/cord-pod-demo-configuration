## Demo CORD POD Configuration

The file, `racks.json` contains the configuration information for the CORD POD demo racks. This includes the information required to configure network connectivity, CORD leaf-spine fabric, XOS, etc. and is used to drive some of the automation around the demo PODS.

## Leaf-Spine Fabric
CORD's leaf-spine fabric is implemented using the segment routing capability of ONOS. There are some key learnings that need to be understood when building out and deploying the segment routing capability:

- ONOS today (10-MAR-2016) does not support hosts that are dual homed to multiple leaves.
- The hosts behind a leaf must be in the same subnet
- The leaf routers mimic a gateway on their subnet, typically numbered `x.x.x.254`. This means the `ARP` for that address and should be the default gateway for the fabric on the hosts.
- Each host needs a route added to it that defines a route to the other leaf subnets through its default gateway (i.e., `x.x.x.254`).
    _eg, on a host on subnet `10.3.1.0/24`, with the other leaf on subnet `10.3.2.0/24`_

        route add -net 10.3.2.0/24 gw 10.3.1.254

- Each switch has an open flow ID. This is in the `JSON` data document, but for reference they are:

    ```
    spine-1 : of:0000000000000011
              0x0000000000000011
    spine-2 : of:0000000000000012
              0x0000000000000012
    leaf-1  : of:0000000000000021
              0x0000000000000021
    leaf-2  : of:0000000000000022
              0x0000000000000022
    ```
- The basic flow is that when traffic enters the leaf it is evaluate by OF table 10, which matches untagged packets. If the packet is untagged, a rule adds a tag 4093 and then the packet evaluation continues on table 10. Another rule is matched against tag 4093 where it is then forwarded to table 20. Table 20 attempts to match against the destination MAC and if matched another tag is added and evaluation continues in table 30. Eventually things get unwrapped at the other end as destination MACs are matched to output ports.
    - You can view specific table rules by using the `flows` command from the ONOS CLI. Use `flows --help` from the CLI to see the full options for the command, but to see the flows for table 10 on leaf-1 in short mode you can use:
    ```
    onos> flows -s added of:0000000000000021 20
    ```
    _Remember you can pipe the output of an ONOS command `grep` to get a colored match results and `more` to page through the output._

### Process to Run Fabric
__NOTE:__ _only a single instance of the fabric can be run at a time, thus if multiple people want to play they must play sequentially, not concurrently_

1. Clear and reset the switches
    - Log in to each witch and make sure the openflow agent (`indigo`) is not running. The actual process is named `brcm-indigo-ofdpa-ofagent`. If it is running `kill -9` it. On our switches there is a script in the `$HOME` directory called `killit` that will do this for you.
    - Purse the ASICs. This can be done via the command `/usr/bin/ofdpa-i.12.1.1/examples/client_cfg_purge` or the script `purge` in the `$HOME` directory.
    - If at any time the switch seems the hang at the command prompt you will need to restart the `ofpda` service, `service ofpda restart`.
2. Clear and reset the ONOS segment routing configuration.
    - If ONOS isn't running, start it up. You should be using the latest version of ONOS, off the `master` branch, `cloned` and built from `https://gerrit.onosproject.org/onos`.
    - Verify that the required applications for segement routing have been activated using the `apps -a -s` command at the ONOS prompt.

        ```
        onos> apps -a -s
        *   9 org.onosproject.lldpprovider         1.5.0.SNAPSHOT ONOS LLDP link provider.
        *  26 org.onosproject.hostprovider         1.5.0.SNAPSHOT ONOS host location provider.
        *  27 org.onosproject.openflow-base        1.5.0.SNAPSHOT OpenFlow protocol southbound providers
        *  28 org.onosproject.openflow             1.5.0.SNAPSHOT OpenFlow southbound meta application
        *  41 org.onosproject.drivers              1.5.0.SNAPSHOT Default device drivers
        *  56 org.onosproject.segmentrouting       1.5.0.SNAPSHOT Segment routing application.
        ```

    If the applications are not activated, they can be activated with the command `app activate <name>`.
    - Delete any existing network configuration from ONOS. This can be done via the command `curl -slL -X DELETE --header "Accept: application/json" "http://karaf:karaf@localhost:8181/onos/v1/network/configuration"`. _You may have to replace the username/password credentials and hostname in the curl command to fit your environment._
    - Verify the configuration was removed via `curl -sLl -X GET --header "Accept: application/json" "http://karaf:karaf@localhost:8181/onos/v1/network/configuration"`.
    - Push the network configuration that works with segment routing. Currently (10-MAR-2016) that can be found at `https://raw.githubusercontent.com/ciena/cord-onos-demo/play/rack-2-network-cfg/network-cfg.json`. This is a JSON document with comments, so before you can push this to ONOS you will need to strip the comments from the file. To push this document to ONOS you can use `cat network-cfg.json | sed -e 's|//.*$||g' | curl -slL -X POST --header "Content-Type: application/json" --header "Accept: application/json" -d "@-" "http://karaf:karaf@localhost:8181/onos/v1/network/configuration"`.
    - Verify the configuration was accepted by ONOS with `curl -sLl -X GET --header "Accept: application/json" "http://karaf:karaf@localhost:8181/onos/v1/network/configuration"`.
    - Exit and restart ONOS. __This is required because, supposedly the current (10-MAR-2016) version of segment routing will only evaluate the network configuration when the application starts and the application is automatically started when ONOS starts.__
3. Start the OFDPA agent on the switches.
    - _I typically do this spine-1, spine-2, leaf-1, leaf-2; but I don't think order really matters._
    - The switches should come up clear, i.e. no error messages. If this is not the case then something is wrong and you likely have to start over from the beginning. __The solution does not recover from errors well and typically means a complete restart of the process.__
    - Verify that the flows have been pushed to the switches. You can do this via the ONOS CLI command `flows pending_add`.

        ```
        onos> flows pending_add
        deviceId=of:0000000000000011, flowRuleCount=0
        deviceId=of:0000000000000012, flowRuleCount=0
        deviceId=of:0000000000000021, flowRuleCount=0
        deviceId=of:0000000000000022, flowRuleCount=0
        ```

    You can view the number of flows pushed to each device using the `flows -c` command for flow count.

        ```
        onos> flows -c
        deviceId=of:0000000000000011, flowRuleCount=14
        deviceId=of:0000000000000012, flowRuleCount=14
        deviceId=of:0000000000000021, flowRuleCount=39
        deviceId=of:0000000000000022, flowRuleCount=35
        ```

    __NOTE:__ _I would have expected that `of:22` and `of:21` would have had the same count, this is a concern._
4. Ping Away
    - At this point you should be able to `ssh` to one of the hosts in the POD and ping to the other over the fabric. For example if you are on `cord-r1-s4` your should be able to `ping` host `cord-r2-s4`'s fabric address of `10.3.2.2`. The fabric IPs for hosts are:
        ```
        cord-r2-s1 : 10.3.1.1
        cord-r2-s2 : 10.3.1.2
        cord-r2-s3 : 10.3.2.1
        cord-r2-s4 : 10.3.2.2
        ```
5. When things go wrong, and they will
    - A useful debugging tool is to increase the debug log output for segment routing in ONOS. This can be done from the ONOS command line prompt:

        ```
        onos> log:set TRACE org.onosproject.segmentrouting
        ```

    After setting this value you can tail the log to see additional information as ping requests are made:
    
        ```
        onos> log:tail
        ```
