# Migrating from Openvswitch to OVN

## Introduction

The migration has been done on a running Openstack infrastructure. There were a few VMs running, that
could have downtimes and network failures during this procedure.

All hosts are in Ubuntu 22.04 LTS, and Openstack Yoga.

## Infrastructure architecture

* There are 2 hosts that have haproxy, maxscale and keepalived.
  * haproxy:
    * frontends expose the Openstack dashboard and APIs to the public
    * terminate ssl to 3 hosts that are Openstack controllers
  * maxscale: distribute database calls to a mariadb+galera cluster with 3 hosts
  * keepalived: Public VirtualIP between the 2 hosts.

* '`cloud_controllers`' There are 3 hosts running the Openstack dashboard and APIs, are pointed to by the above-mentioned
haproxy backend.

* '`cloud_network`' There are 2 hosts that run initially all neutron agents: DHCP, L3, Metadata and Openvswitch.
  * L3 routers in HA

* '`cloud_compute`' Several compute nodes running nova-compute, libvirt/kvm, neutron agent openvswitch and neutron agent sriov.

* CEPH cluster or block and object storage.

## Ansible playbooks for migration

The procedure is largely based on this playbook:
<https://github.com/openstack/neutron/blob/master/tools/ovn_migration/migrate-to-ovn.yml>.

Run the installation playbook, note that this only has the packages concerning OVN and neutron:

```bash
ansible-playbook 01-migrate-to-ovn-install.yml
```

Run the config playbook to copy the configuration files to the hosts, the relevant parts of
configuration files are in the `conf` directory:

```bash
ansible-playbook 02-conf.yml
```

* Note in the ml2_conf.ini the variables `ovn_nb_connection` and `ovn_sb_connection` have the IPs of the 3
controller hosts because they all run the OVN northbound and southbound databases.

* Note in the neutron.conf the `oslo_messaging_rabbit` for rabbitmq quorum queues

## The old bridges

**This part is disruptive and is not sure to be correct. The VMs network will stop work at this point.
In principle, they will recover since all information about the neutron network, routers, ports is in the database.**

Remove all ports from the br-int on the compute nodes and network nodes

```bash
for pt in `ovs-vsctl list-ports br-int`
do
  ovs-vsctl del-port br-int $pt
done
```

Also delete all ports from br-ex (on the network nodes) and br-vlan (on the compute nodes) that do not correspond to physical interfaces

```bash
ovs-vsctl del-port br-ex phy-br-ex
```

## Remaining playbooks

Now you can run:

```bash
ansible-playbook 03-migrate-to-ovn-setup-northd.yml
ansible-playbook 04-migrate-to-ovn-setup-contr.yml
```

Some or all tasks maybe better to execute them manually.

## The OVN nb and sb cluster

To setup a cluster of the databases (northbound and southbound), on the openstack controller nodes stop
neutron-server and ovn-central on all nodes:

```bash
systemctl stop ovn-central
rm -rvf /var/lib/ovn
```

Start the ovn-central on all controllers one at a time

```bash
systemctl restart ovn-central
```

Migrate the neutron database from OVS to OVN, first stop the neutron-server:

```bash
## In the playbook
neutron-ovn-db-sync-util --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini \
  --log-file /var/log/neutron/neutron-db-sync.log --ovn-neutron_sync_mode migrate --debug

neutron-ovn-db-sync-util --config-file /etc/neutron/neutron.conf \
 --config-file /etc/neutron/plugins/ml2/ml2_conf.ini --ovn-neutron_sync_mode repair --debug
```

Restart the neutron-server.

Check and/or set in the ovn-controllers that

```bash
ovs-vsctl set-manager ptcp:6640:127.0.0.1
```

Also if needed in the network nodes:

```bash
ovs-vsctl del-controller br-vlan
ovs-vsctl del-controller br-int
ovs-vsctl del-controller br-ex
ovs-vsctl set-controller br-vlan tcp:127.0.0.1:6640
ovs-vsctl set-controller br-int tcp:127.0.0.1:6640
ovs-vsctl set-controller br-ex tcp:127.0.0.1:6640
systemctl restart openvswitch-switch ovn-host
```

The prefered network type is now `geneve` instead of `vxlan`. Check the neutron database:

```bash
MariaDB [neutron]> select * from networksegments;
+--------------------------------------+--------------------------------------+--------------+------------------+-----------------+------------+---------------+------------------+------+
| id                                   | network_id                           | network_type | physical_network | segmentation_id | is_dynamic | segment_index | standard_attr_id | name |
+--------------------------------------+--------------------------------------+--------------+------------------+-----------------+------------+---------------+------------------+------+
| 84334795-0a9c-46dc-bb45-abd858a787ae | 282e63e3-5120-4396-a63d-0186e5e96466 | vxlan        | NULL             |             942 |          0 |             0 |               27 | NULL |
| 8c866571-b041-426c-9ddd-5f126fd694e3 | ab3f0f85-a509-406a-8dca-5db13fbcb48b | vxlan        | NULL             |             230 |          0 |             0 |               21 | NULL |
| 949f5d71-fd93-45e3-9895-ad2415541e89 | 9e151884-67a5-4905-b157-f08f1b3b0040 | vxlan        | NULL             |              10 |          0 |             0 |              144 | NULL |
| b4f4131d-f988-49ef-9440-361d747af8eb | 12a0ab09-d130-4e69-9aa2-c28c66509b02 | vxlan        | NULL             |             665 |          0 |             0 |               33 | NULL |
| ccfefb4d-0da0-4138-ac74-be1934eca9d7 | 3fb2d48e-8c71-4bca-92ce-f64a4c932338 | vlan         | vlan             |             200 |          0 |             0 |              126 | NULL |
| fc4d896e-9eb8-4a73-a363-223a5dc81ec5 | dddfdce8-a8fd-4802-a01c-261b92043488 | vlan         | vlan             |             100 |          0 |             0 |              120 | NULL |
+--------------------------------------+--------------------------------------+--------------+------------------+-----------------+------------+---------------+------------------+------+

MariaDB [neutron]> update networksegments set network_type='geneve' where network_type='vxlan';
Query OK, 4 rows affected (0.008 sec)
Rows matched: 4  Changed: 4  Warnings: 0
```

## Post migration

List and remove old network neutron agents:

```bash
openstack network agent list
openstack network agent list -c ID -f value > netagents  # Edit and remove the OVN agents
for agent in `cat netagents`; do echo $agent; openstack network agent show $agent; done
for agent in `cat netagents`; do echo $agent; openstack network agent set --disable $agent; done
for agent in `cat netagents`; do echo $agent; openstack network agent delete $agent; done
```

On the network nodes (former neutron agents)

```bash
ovs-vsctl list interface | awk '/name[ ]*: qr-|ha-|qg-|rfp-|sg-|fg-/ { print $3 }' | xargs -n1 ovs-vsctl del-port

for pt in `ovs-vsctl list port|awk -F': ' '{print $2}' | grep tap`
do
  ovs-vsctl del-port $pt
done

apt purge neutron-dhcp-agent neutron-l3-agent neutron-metadata-agent neutron-openvswitch-agent
apt -y autoremove
```

Check if the hostname is the FQDN

```bash
ovs-vsctl get open_vswitch . external-ids:hostname
ovs-vsctl set open_vswitch . external-ids:hostname=net_node01.pt
```

## Summary

### Openstack controller nodes

On the controller nodes, the following package should be installed:

* ovn-central - dependencies:
  * ovn-common
  * openvswitch-common
  * Provides:
    * ovn-central.service
    * ovn-northd.service
    * ovn-ovsdb-server-nb.service
    * ovn-ovsdb-server-sb.service

The neutron-server package is already installed.

The services running in the controllers are (only regarding neutron and OVN):

* neutron-server
* ovn-central
  * ovn-northd
  * ovn-ovsdb-server-nb
  * ovn-ovsdb-server-sb

### Network nodes - former neutron agents nodes

The network neutron agents node is now called the OVN Gateway Chassis, the same packages as the nova compute
nodes, should be installed:

* ovn-host - dependencies:
  * openvswitch-common
  * openvswitch-switch
  * ovn-common
  * python3-openvswitch

The services running in the network nodes are:

* ovn-host
  * ovn-controller


Reference here: <https://docs.openstack.org/openstack-ansible-os_neutron/latest/app-ovn.html>.
Is capable of providing external (north/south) connectivity to the tenant traffic. This is essentially a node(s) capable
of hosting the logical router performing SNAT and DNAT (Floating IP) translations. East/West traffic flow is not limited
to a gateway chassis and is performed between an OVN chassis nodes.

*When planning out your architecture, it is important to determine early if you want to centralize OVN gateway chassis functions to a subset of nodes or across all compute nodes. Centralizing north/south routing to a set of dedicated network or gateway nodes is reminiscent of the legacy network node model. Enabling all compute nodes as gateway chassis will narrow the failure domain and potential bottlenecks at the cost of ensuring the computes can connect to the provider networks.*

### Nova compute nodes

On the nova-compute nodes the following packages should be installed:

* ovn-host - dependencies:
  * openvswitch-common
  * openvswitch-switch
  * ovn-common
  * python3-openvswitch

* neutron-ovn-metadata-agent - dependencies:
  * neutron-common
  * python3-os-ken
  * python3-neutron-lib
  * python3-neutronclient
  * python3-ovsdbapp
  * python3-os-vif
  * python3-neutron

The services running in the compute nodes are:

* neutron-ovn-metadata-agent
* ovn-host
  * ovn-controller

## Regarding L3HA

On the ml2_conf.ini

enable_distributed_floating_ip
Type:
boolean

Default:
False

Enable distributed floating IP support. If True, the NAT action for floating IPs will be done locally and not in the centralized gateway.
This saves the path to the external network. This requires the user to configure the physical network map (i.e. ovn-bridge-mappings) on each compute node.

## Documentation

* <https://www.jimmdenton.com/migrating-lxb-to-ovn/>
* <https://docs.openstack.org/openstack-ansible-os_neutron/latest/app-ovn.html>
* <https://docs.redhat.com/it/documentation/red_hat_openstack_platform/15/html-single/networking_with_open_virtual_network/index>
* <https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/wallaby/app-ovn.html>
* <https://docs.openstack.org/neutron/latest/ovn/faq/index.html>
* <https://docs.redhat.com/en-us/documentation/red_hat_openstack_platform/17.1/pdf/migrating_to_the_ovn_mechanism_driver/Red_Hat_OpenStack_Platform-17.1-Migrating_to_the_OVN_mechanism_driver-en-US.pdf>
