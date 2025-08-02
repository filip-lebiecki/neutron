# [Youtube](https://youtu.be/JyYZ1gOig64?si=K8JbLzjVA5ZXPue8)

## Cloud Networking
OpenStack Neutron is the networking service that creates and manages a virtualized networking framework. It uses familiar concepts like routers, switches, and subnets, but all are implemented in software.

**Commands**

This command lists the OpenStack services that are running. Neutron is one of the core services.

* `openstack service list`

## Neutron Architecture
The Neutron server is the brain of the operation, receiving network requests and acting as a central controller. Its architecture is flexible, using a plugin system like the Modular Layer 2 (ML2) plugin and agents that perform the actual networking configuration on each host.

**Commands**

This command verifies that the Neutron server is running inside a Docker container.

* `docker ps | grep neutron-server`

This command displays the contents of the ML2 plugin configuration file.

* `docker exec -it neutron_server cat /etc/neutron/plugins/ml2/ml2_conf.ini`

Lists the various Neutron agents, such as the Open vSwitch agent, DHCP agent, and L3 agent, which are the distributed workers that configure networking on each host.

* `openstack network agent list`

Another command to list the running Neutron agents, which also run as Docker containers.

* `docker ps | grep neutron`

Confirms that the OVS (Open vSwitch) components are running on a compute node. The OVS agent is responsible for creating and managing the local virtual switch.

* `docker ps | grep openvswitch`

## OVS firewall driver
The OVS agent is responsible for enforcing security group rules. The firewall driver can be configured for `iptables` (hybrid mode) or `openvswitch` (native mode). Native mode is more efficient as it implements security group rules directly as OpenFlow rules.

**Commands**

Displays the configuration file for the Open vSwitch agent, where you can see the `firewall_driver` setting.

* `docker exec -it neutron_openvswitch_agent cat /etc/neutron/plugins/ml2/openvswitch_agent.ini`

Creates a dedicated folder to store a custom Neutron configuration file for Kolla Ansible. 

* `mkdir -p /etc/kolla/config/neutron`

Creates and opens a new configuration file to set the `firewall_driver` to `openvswitch`.

* `vi /etc/kolla/config/neutron/openvswitch_agent.ini`

```
[securitygroup]
firewall_driver = openvswitch
```

Runs the Kolla Ansible deployment playbook to apply the new configuration and restart the `neutron-openvswitch-agent` containers.

* `kolla-ansible deploy -i ./multinode`

Verifies that the `firewall_driver` setting was successfully changed to `openvswitch`.

* `docker exec -it neutron_openvswitch_agent cat /etc/neutron/plugins/ml2/openvswitch_agent.ini` 

## Tenant networks vs Provider Networks
Tenant networks are private, project-scoped overlay networks created by users for east-west traffic between VMs. Provider networks are administrator-created networks that bridge directly to an existing physical network for north-south traffic to the outside world.

**Commands**

This command shows that the `ml2_conf.ini` file contains configurations for tenant networks, such as VXLAN encapsulation, which is a common type.

* `docker exec -it neutron_server cat /etc/neutron/plugins/ml2/ml2_conf.ini`

## Network and subnet
An OpenStack network is a distributed Layer 2 virtual switch that acts as a container for your subnets. A subnet is a Layer 3 concept that defines the IP address range for the VMs connected to a network.

**Commands**

Creates a new network named `my-network`.

* `openstack network create my-network`

Creates a new subnet named `my-subnet` within `my-network` with a specified IP address range, gateway, and DNS server.

* `openstack subnet create --network my-network --subnet-range 10.128.0.0/24 --gateway 10.128.0.1 --dns-nameserver 1.1.1.1 my-subnet`

Lists the ports in your project, which are virtual network plugs for VMs.

* `openstack port list`

Shows the device owner for a specific port to identify who or what is using it.

* `openstack port show 35951b74-6bee-466f-8204-bbed0769da39 -c device_owner`

Lists which nodes are running the DHCP agents.

* `openstack network agent list --agent-type dhcp`

Lists the network namespaces on the node, showing where services like DHCP live.

* `ip netns`

Allows you to enter a bash shell within a specific DHCP network namespace to inspect its interfaces and processes.

* `sudo ip netns exec qdhcp-6a880951-2f40-4412-9da2-4d03c79fde59 bash`

Checks the network interfaces inside the namespace.

* `ip a`

Checks for the `dnsmasq` process, which acts as the DHCP server.

* `ps -ef \| grep dnsmasq`

Exits the network namespace.
   
* `Ctrl+D`

Creates a dedicated port for a VM with a static IP address.

* `openstack port create --network my-network --fixed-ip subnet=my-subnet,ip-address=10.128.0.100 vm1-port`

Creates a new VM named `vm1` and attaches it to the port created in the previous step.
    
* `openstack server create --flavor m1.tiny --image cirros --port vm1-port vm1`

Lists all ports with detailed information, including their status and owner.

* `openstack port list --long`

## OVS bridge
An OVS (Open vSwitch) bridge is a software-based virtual switch that connects various network interfaces, allowing them to communicate. Standard OpenStack deployments use three main bridges on each network node: the Integration Bridge (br-int), the Tunnel Bridge (br-tun), and the External Bridge (br-ex).

**Commands**

Displays the configuration of the OVS bridges and patch ports on a network node. 

* `docker exec -it openvswitch_db ovs-vsctl show`

## Neutron ports
A Neutron port is a VM's virtual network interface that holds its IP address, MAC address, and security group settings. Creating a port manually before creating a VM gives you more control and flexibility.

**Commands**

Creates a VM, allowing OpenStack to automatically create a port for it.

* `openstack server create --flavor m1.tiny --image cirros --network my-network vm2`

Lists the ports that are owned by a Nova compute instance. 

* `openstack port list --device-owner compute:nova`

Lists all the servers (VMs) in your project.

* `openstack server list`

Deletes the VM named `vm2`.
    
* `openstack server delete vm2`

After deleting a VM with an automatically created port, this command shows that the port is also deleted.

* `openstack port list --device-owner compute:nova`

Deletes a VM that had a manually created port.

* `openstack server delete vm1`

After deleting a VM with a manually created port, this command shows that the port remains, but is no longer attached.
   
* `openstack port list`

Manually deletes the port that was not deleted when its associated VM was removed.

* `openstack port delete vm1-port`

## Communication
VMs on the same subnet can communicate with each other even if they are on different physical compute nodes, thanks to overlay networks created by Open vSwitch using technologies like VXLAN.

**Commands**

Creates a VM named `vm1` on the compute node `stack1`.

* `openstack server create --flavor m1.tiny --image cirros --network my-network --availability-zone nova:stack1 vm1`

Creates a VM named `vm2` on the compute node `stack2`.

* `openstack server create --flavor m1.tiny --image cirros --network my-network --availability-zone nova:stack2 vm2`

Confirms the physical host where `vm1` is running.

* `openstack server show vm1 -c OS-EXT-SRV-ATTR:host`

Confirms the physical host where `vm2` is running.
    
* `openstack server show vm2 -c OS-EXT-SRV-ATTR:host`

Lists all servers and their IP addresses.

* `openstack server list`

Finds the name of the VM on the underlying operating system.

* `sudo virsh list --all`

Attaches to the console of a specific VM instance.

* `sudo virsh console instance-00000013`

Pings the IP address of `vm2` from within `vm1`'s console to test connectivity between the two VMs.

* `ping 10.128.0.175`

Commands to exit the virsh console.

* `Ctrl+C, Ctrl + \`

Shows the OVS configuration on `stack1`, highlighting the VXLAN port that enables cross-node communication.

* `docker exec -it openvswitch_db ovs-vsctl show`

## Two subnets
By default, subnets within the same network are isolated from each other. They cannot communicate without a router.

**Commands**

Creates a second subnet named `my-subnet-2` in the same network `my-network`.

* `openstack subnet create --network my-network --subnet-range 10.129.0.0/24 --gateway 10.129.0.1 --dns-nameserver 1.1.1.1 my-subnet-2`

Lists all the subnets to confirm both are present.

* `openstack subnet list`

Creates a new port specifically for the second subnet.

* `openstack port create --network my-network --fixed-ip subnet=my-subnet-2 vm3-port`
  
Creates a new VM named `vm3` and attaches it to the `vm3-port`.

* `openstack server create --flavor m1.tiny --image cirros --port vm3-port vm3`
 
Lists the servers to confirm that `vm3` is in the `10.129` subnet.

* `openstack server list`

Attempts to ping a VM in the second subnet from a VM in the first subnet to demonstrate that communication fails by default.

* `ping 10.129.0.93`

Shows the routing table on `vm1` to confirm that the route to `vm3` points to the default gateway.

* `ip route get 10.129.0.93`

Pings the default gateway from within `vm1` to show that it also fails, as it is isolated from the second subnet.

* `ping 10.128.0.1`

## Router
An OpenStack router is a virtual Layer 3 device that connects different Neutron networks and enables communication between their subnets.

**Commands**

Creates a new OpenStack router named `my-router`.

* `openstack router create my-router`

Attaches `my-subnet` to `my-router`.

* `openstack router add subnet my-router my-subnet`

Attaches `my-subnet-2` to `my-router`.
  
* `openstack router add subnet my-router my-subnet-2`

Attempts to ping `vm3` from `vm1` again, now that the router is in place. [cite_start]The ping should now succeed.

* `ping 10.129.0.93`

Shows that the routing table now correctly points to the new router's gateway. 

* `ip route get 10.129.0.93`

Pings the default gateway from `vm1` to confirm it is now reachable.

* `ping 10.128.0.1`

Lists the network namespaces to show the new `qrouter` namespace for the router. 

* `ip netns ls`

Enters a bash shell within the router's network namespace to inspect its configuration.
   
* `sudo ip netns exec qrouter-761cdf8d-c53f-4b5f-a397-770b822d0327 bash`
  
Examines the network interfaces within the router's namespace, which will show two "QR" interfaces.

* `ip a`

Shows the routing table within the router's namespace, confirming it's a central point for inter-subnet communication.

* `ip r`

Inspects the OVS integration bridge to see that the router's interfaces are plugged into it.

* `docker exec -it openvswitch_db ovs-vsctl show`

## Second network
A router can connect subnets from different networks, not just those within the same network.

**Commands**

Creates a second network named `my-network-2`.

* `openstack network create my-network-2`

Creates a subnet named `my-subnet-3` within the new network. 

* `openstack subnet create --network my-network-2 --subnet-range 10.130.0.0/24 --gateway 10.130.0.1 --dns-nameserver 1.1.1.1 my-subnet-3`

Creates a VM named `vm4` in `my-network-2`.
   
* `openstack server create --flavor m1.tiny --image cirros --network my-network-2 vm4`

Attaches `my-subnet-3` from the new network to `my-router`.
 
* `openstack router add subnet my-router my-subnet-3`

## External network
An external network allows VMs to access the public internet. This is done by attaching a provider network to the router and setting it as the external gateway.

**Commands**

Creates a provider network named `public` and marks it as external. This network bridges to the physical network.

* `openstack network create --external --provider-physical-network physnet1 --provider-network-type flat public`

Shows the ML2 plugin configuration, which declares `physnet1` as a valid flat network. 

* `docker exec -it neutron_server cat /etc/neutron/plugins/ml2/ml2_conf.ini`

Shows that the provider network `physnet1` is associated with the `br-ex` bridge in the OVS agent.

* `docker exec -it neutron_openvswitch_agent cat /etc/neutron/plugins/ml2/openvswitch_agent.ini`

Shows that the physical interface `eth1` is connected to the external bridge.

* `docker exec -it openvswitch_db ovs-vsctl show`

Creates a subnet for the public network with a specific IP range and gateway.

* `openstack subnet create --no-dhcp --subnet-range 192.168.12.0/24 --network public --allocation-pool 'start=192.168.12.100,end=192.168.12.120' --gateway 192.168.12.200 public-subnet`

Attaches `my-router` to the public network and sets it as the gateway.

* `openstack router set --external-gateway public my-router`

Shows the details of the router, confirming that it has an IP assigned from the public network and that SNAT is enabled.

* `openstack router show my-router`

Pings a public DNS server from within a VM's console to test internet connectivity.

* `ping dns.google`

## Floating IPs
A Floating IP is a public IP address from the external network that can be dynamically associated with a VM's private IP. This allows the VM to be exposed to the outside world without compromising the security of the private network.

**Commands**

Allocates a Floating IP from the public network pool.

* `openstack floating ip create public`

Associates the newly created Floating IP with `vm1`. 

* `openstack server add floating ip vm1 192.168.12.101`

Lists the Floating IPs and their associations.

* `openstack floating ip list`

Lists the servers, and shows their associated Floating IPs.

* `openstack server list`
