# Migration procedure documentation

## Based on Redhat Tripleo

Official github neutron repository <https://github.com/openstack/neutron> branch stable/2024.2 directory - tools/ovn_migration

playbooks/ovn-migration.yml

* Install packages on the nodes (see next sections)

* **NOT NEEDED** name: Backup controllers pre-migration - hosts: localhost
  roles - recovery-backup

* **NOT NEEDED** name: Pre migration and validation tasks - hosts: localhost
  roles - pre-migration

* **NOT NEEDED** name: Backup tripleo container config files on the nodes - hosts: ovn-controllers
  roles - backup

* name: Stop ml2/ovs resources - hosts: ovn-controllers
  roles - stop-agents
  **Mandatory**: stop and disable **neutron-openvswitch-agent neutron-dhcp-agent neutron-l3-agent neutron-metadata-agent**

* name: Set up OVN and configure it using tripleo - hosts: localhost
  roles - tripleo-update
  **May be it's necessary** to be checked later

* name: Do the DB sync and dataplane switch - hosts: ovn-controllers, ovn-dbs
  roles - migration

  * tunnel_bridge: "br-tun"
  * ovn_bridge: "br-int"

  * Tasks:
    * clone-dataplane.yml
      * recreate_bridge_mappings
        * `ovn_orig_bm=ovs-vsctl get open . external_ids:ovn-bridge-mappings`
        * `ovs-vsctl set open . external_ids:ovn-bridge-mappings-back="$ovn_orig_bm"`
        * new_mapping
        * `ovs-vsctl set open . external_ids:ovn-bridge-mappings="$new_mapping"`
      * restart ovn_controller
      * copy_interfaces_to_br_migration
        * interfaces=$(ovs-vsctl list-ifaces br-int | egrep -v 'qr-|ha-|qg-|rfp-|sg-|fg-')
        * ifmac=$(ovs-vsctl get Interface $interface external-ids:attached-mac)
        * ifstatus=$(ovs-vsctl get Interface $interface external-ids:iface-status)
        * ifid=$(ovs-vsctl get Interface $interface external-ids:iface-id)
        * ifname=x$interface
        * ovs-vsctl -- --may-exist add-port $OVN_BR_MIGRATION $ifname -- set Interface $ifname type=internal -- set Interface $ifname external-ids:iface-status=$ifstatus -- set Interface $ifname external-ids:attached-mac=$ifmac -- set Interface $ifname external-ids:iface-id=$ifid

    * sync-dbs.yml
      * systemctl stop neutron-server
      * neutron-ovn-db-sync-util --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini --log-file /var/log/neutron/neutron-db-sync.log --ovn-neutron_sync_mode migrate --debug
      * systemctl start neutron-server

    * activate-ovn.yml
      * stop ovn_controller
      * restore bridge mappings
      * Reset OpenFlow protocol version before ovn-controller takes over: `ovs-vsctl set Bridge {{ ovn_bridge }} protocols=[]`
      * ovs-vsctl set-fail-mode $bridge standalone
      * ovs-vsctl set Bridge $bridge protocols=[]
      * ovs-vsctl del-controller $bridge
      * Delete controller from integration bridge: `ovs-vsctl del-controller {{ ovn_bridge }}`
      * Activate ovn-controller by configuring integration bridge: `ovs-vsctl set open . external_ids:ovn-bridge={{ ovn_bridge }}`
      * start ovn_controller
      * Delete ovs bridges - br-tun and br-migration
        * ovs-vsctl --if-exists del-br {{ tunnel_bridge }}
        * ovs-vsctl --if-exists del-br br-migration
        * ovs-vsctl --if-exists del-port br-int patch-tun

    * cleanup-dataplane.yml
      * Cleanup neutron router and dhcp interfaces: ovs-vsctl list interface | awk '/name[ ]*: qr-|ha-|qg-|rfp-|sg-|fg-/ { print $3 }' | xargs -n1 ovs-vsctl del-port
      * dhcp tap ports: ovs-vsctl del-port $dhcp_port
      * Check if neutron trunk subports need cleanup:
        * ovs-vsctl list interface | awk '/name[ ]*: sp[it]-/ { print $3 }' | grep 'spi-\|spt-'
        * ovs-vsctl list interface | awk '/name[ ]*: sp[it]-/ { print $3 }' | xargs -n1 ovs-vsctl del-port
      * Clean neutron datapath security groups from iptables: {{ iptables_exec }}-save > /tmp/iptables-before-cleanup
      * Cleanup neutron datapath resources
      * Cleanup Neutron ml2/ovs namespaces: ip netns delete $netns

* name: Post migration - hosts: localhost
  roles - delete-neutron-resources, post-migration
  * openstack network agent delete $i
  * openstack port delete $p
  * openstack network delete $i

* **NOT NEEDED** name: Final validation - hosts: localhost
  roles - post-migration

## Based on ansible role openstack-neutron

The repository is <https://github.com/openstack/openstack-ansible-os_neutron>, tasks/main.yml

* Install packages on the nodes (see next sections)

Tasks:

* main.yml
  * **NOT NEEDED** **neutron_check.yml** - check some ansible variables.
  * **NOT NEEDED** **{{ neutron_install_method }}_install.yml** - installation method: we will use deb packages.
  * **NOT NEEDED** role: openstack.osa.db_setup - database setup and creation was already done.
  * **NOT NEEDED** role: openstack.osa.mq_setup - rabbitmq setup and config already done.
  * **NOT NEEDED** Create the neutron provider networks fact
  * **NOT NEEDED** Importing neutron_pre_install tasks: **neutron_pre_install.yml**: creation of group, user, directories and files

  * **NOT NEEDED** Importing neutron_install tasks: **neutron_install.yml**: installation
    * Remove known problem packages: conntrackd in the case of Ubuntu
    * **NOT NEEDED** Include FRR role for OVN BGP Agent role: frrouting
    * **NOT NEEDED** Retrieve the constraints URL: ceilometer stuff
    * **NOT NEEDED** Install the python venv: source install
    * **NOT NEEDED** Initialise the upgrade facts: only openstack ansible matters
    * **NOT NEEDED** Importing neutron_apparmor tasks

  * **NOT NEEDED** Create and install SSL certificates for API
  * **NOT NEEDED** Create and install SSL certificates for OVN: we will try to use non ssl protocols - tcp

  * Include provider specific config(s) - tasks: **ovn_config.yml**
    * **NOT NEEDED** Configure ovn-controller: template ovn-controller-opts.j2 - possible not needed since these are options for SSL certificates
    * **NOT NEEDED** Mask setting OVS hostname service: bug in ovs-vsctl in antelope
    * **cloudcont** Ensure ovn-northd service is started and enabled: `systemctl start ovn-central`
      * Starts all services: ovn-northd, ovn-ovsdb-server-nb, ovn-ovsdb-server-sb
    * **cloudcomp,cloudnet** Ensure ovn-controller service is started and enabled: `systemctl start ovn-host`
      * Starts as well ovn-controller

    * Including setup_ovs_ovn tasks: **setup_ovs_ovn.yml**
      * **cloudcomp** Set openvswitch hostname: `ovs-vsctl set open_vswitch . external-ids:hostname='neutron-ovn-controller'`
      * **cloudnet** Set CMS Options for Gateway Scheduling: `ovs-vsctl set Open_vSwitch . external-ids:ovn-cms-options=enable-chassis-as-gw,availability-zones={{ neutron_availability_zone = nova }}"`
      * **cloudcomp** Configure OVN Southbound Connection: `ovs-vsctl set open . external-ids:ovn-remote={{ neutron_ovn_sb_connection }}`
      * **cloudcomp** Configure Supported OVN Overlay Protocols: `ovs-vsctl set open . external-ids:ovn-encap-type={{ network_types = ... }}` - **ONLY** "geneve, vxlan"
      * **cloudcomp** Configure Encapsulation Address for Overlay Traffic: `ovs-vsctl set open . external-ids:ovn-encap-ip={{ neutron_local_ip }}`
      * **cloudcomp** Register existing OVSDB Manager(s): `ovs-vsctl get-manager`
      * **cloudcomp** Create OVSDB Manager: `ovs-vsctl --id @manager create Manager "target=\"{{ neutron_ovsdb_manager }}\"" -- add Open_vSwitch . manager_options @manager`
      * **cloudcomp,cloudnet** Setup Network Provider Bridges: openvswitch_bridge {{ neutron_provider_networks.network_mappings }} - this are our variables neutron_comp_ovs_bridge_mappings neutron_net_ovs_bridge_mappings
      * **NOT NEEDED** Add ports to Network Provider Bridges  When ovn-bgp-agent is in use.
      * **cloudcomp** Set the OVN Bridge Mappings in OVS: `ovs-vsctl set open . external-ids:ovn-bridge-mappings={{ neutron_comp_ovs_bridge_mappings }}`
      * **cloudnet** Set the OVN Bridge Mappings in OVS: `ovs-vsctl set open . external-ids:ovn-bridge-mappings={{ neutron_net_ovs_bridge_mappings }}`

      * **cloudcont** Including ovn_cluster_setup tasks: **ovn_cluster_setup.yml** - adapt to our case, or configure manually.

  * Importing neutron_post_install tasks: neutron_post_install.yml
    * **NOT NEEDED** Create plugins neutron dir: already done
    * **ALL nodes** Copy extra neutron rootwrap filters: files/rootwrap.d/ovn-plugin.filters - this will be later incorporated in the 05-conf.yaml
    * **ALL nodes** Copy common neutron config: neutron.conf, ml2_conf.ini - this will be later incorporated in the 05-conf.yaml
    * **NOT NEEDED** Implement policy.yaml if there are overrides configured.
    * **NOT NEEDED** Remove legacy policy.yaml file.
    * **NOT NEEDED** Create symlink to neutron-keepalived-state-change.
    * **NOT NEEDED** Preserve original configuration file(s)
    * **NOT NEEDED** Fetch override files
    * **NOT NEEDED** Copy common neutron config.
    * **NOT NEEDED** Cleanup fetched temp files
    * **NOT NEEDED** Copy neutron ml2 plugin config
    * **cloudnet** Stop haproxy service on debian derivatives with standalone network nodes: systemctl stop haproxy, and disable.

  * Disable dhcp, metadata, openvswitch and l3 agents for OVN scenario.
  * Including neutron_db_setup role: neutron_db_setup.yml
    * **cloudctl_01** neutron-ovn-db-sync-util --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini --log-file /var/log/neutron/neutron-db-sync.log --ovn-neutron_sync_mode migrate --debug
