.. _refarch-provider-networks:

Provider networks
-----------------

A provider (external) network bridges instances to physical network
infrastructure that provides layer-3 services. In most cases, provider networks
implement layer-2 segmentation using VLAN IDs. A provider network maps to a
provider bridge on each compute node that supports launching instances on the
provider network. You can create more than one provider bridge, each one
requiring a unique name and underlying physical network interface to prevent
switching loops. Provider networks and bridges can use arbitrary names,
but each mapping must reference valid provider network and bridge names.
Each provider bridge can contain one ``flat`` (untagged) network and up to
the maximum number of ``vlan`` (tagged) networks that the physical network
infrastructure supports, typically around 4000.

Creating a provider network involves several commands at the host, OVS,
and Networking service levels that yield a series of operations at the
OVN level to create the virtual network components. The following example
creates a ``flat`` provider network ``provider`` using the provider bridge
``br-provider`` and binds a subnet to it.

Create a provider network
~~~~~~~~~~~~~~~~~~~~~~~~~

#. On each compute node, create the provider bridge, map the provider
   network to it, and add the underlying physical or logical (typically
   a bond) network interface to it.

   .. code-block:: console

      # ovs-vsctl --may-exist add-br br-provider -- set bridge br-provider \
        protocols=OpenFlow13
      # ovs-vsctl set open . external-ids:ovn-bridge-mappings=provider:br-provider
      # ovs-vsctl --may-exist add-port br-provider INTERFACE_NAME

   Replace ``INTERFACE_NAME`` with the name of the underlying network
   interface.

   .. note::

      These commands provide no output if successful.

#. On the controller node, source the administrative project credentials.

#. On the controller node, create the provider network in the Networking
   service. In this case, instances and routers in other projects can use
   the network.

   .. code-block:: console

      $ openstack network create --external --share \
        --provider-physical-network provider --provider-network-type flat \
        provider
      +---------------------------+--------------------------------------+
      | Field                     | Value                                |
      +---------------------------+--------------------------------------+
      | admin_state_up            | UP                                   |
      | availability_zone_hints   |                                      |
      | availability_zones        | nova                                 |
      | created_at                | 2016-06-15 15:50:37+00:00            |
      | description               |                                      |
      | id                        | 0243277b-4aa8-46d8-9e10-5c9ad5e01521 |
      | ipv4_address_scope        | None                                 |
      | ipv6_address_scope        | None                                 |
      | is_default                | False                                |
      | mtu                       | 1500                                 |
      | name                      | provider                             |
      | project_id                | b1ebf33664df402693f729090cfab861     |
      | provider:network_type     | flat                                 |
      | provider:physical_network | provider                             |
      | provider:segmentation_id  | None                                 |
      | qos_policy_id             | None                                 |
      | router:external           | External                             |
      | shared                    | True                                 |
      | status                    | ACTIVE                               |
      | subnets                   | 32a61337-c5a3-448a-a1e7-c11d6f062c21 |
      | tags                      | []                                   |
      | updated_at                | 2016-06-15 15:50:37+00:00            |
      +---------------------------+--------------------------------------+

   .. note::

      The value of ``--provider-physical-network`` must refer to the
      provider network name in the mapping.

OVN operations
^^^^^^^^^^^^^^

.. todo: I don't like going this deep with headers, so a future patch
         will probably break this content into multiple files.

The OVN mechanism driver and OVN perform the following operations during
creation of a provider network.

#. The mechanism driver translates the network into a logical switch
   in the OVN northbound database.

   .. code-block:: console

      _uuid               : 98edf19f-2dbc-4182-af9b-79cafa4794b6
      acls                : []
      external_ids        : {"neutron:network_name"=provider}
      load_balancer       : []
      name                : "neutron-e4abf6df-f8cf-49fd-85d4-3ea399f4d645"
      ports               : [92ee7c2f-cd22-4cac-a9d9-68a374dc7b17]

     .. note::

        The ``neutron:network_name`` field in ``external_ids`` contains
        the network name and ``name`` contains the network UUID.

#. In addition, because the provider network is handled by a separate
   bridge, the following logical port is created in the OVN northbound
   database.

   .. code-block:: console

      _uuid               : 92ee7c2f-cd22-4cac-a9d9-68a374dc7b17
      addresses           : [unknown]
      enabled             : []
      external_ids        : {}
      name                : "provnet-e4abf6df-f8cf-49fd-85d4-3ea399f4d645"
      options             : {network_name=provider}
      parent_name         : []
      port_security       : []
      tag                 : []
      type                : localnet
      up                  : false

#. The OVN northbound service translates these objects into datapath bindings,
   port bindings, and the appropriate multicast groups in the OVN southbound
   database.

   * Datapath bindings

     .. code-block:: console

        _uuid               : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        external_ids        : {logical-switch="98edf19f-2dbc-4182-af9b-79cafa4794b6"}
        tunnel_key          : 109

   * Port bindings

     .. code-block:: console

        _uuid               : 8427506e-46b5-41e5-a71b-a94a6859e773
        chassis             : []
        datapath            : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        logical_port        : "provnet-e4abf6df-f8cf-49fd-85d4-3ea399f4d645"
        mac                 : [unknown]
        options             : {network_name=provider}
        parent_port         : []
        tag                 : []
        tunnel_key          : 1
        type                : localnet

   * Logical flows

     .. code-block:: console

        _uuid               : 9af3d8d0-4ddc-4358-baea-608a7f45f0e2
        actions             : "drop;"
        external_ids        : {stage-name="ls_in_port_sec_l2"}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : "eth.src[40]"
        pipeline            : ingress
        priority            : 100
        table_id            : 0

        _uuid               : 4b16b89a-1854-4673-87e8-c7e109d9fda4
        actions             : "drop;"
        external_ids        : {stage-name="ls_in_port_sec_l2"}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : vlan.present
        pipeline            : ingress
        priority            : 100
        table_id            : 0

        _uuid               : 3ae9d916-4b15-4015-bc5a-b688279f4932
        actions             : "next;"
        external_ids        : {stage-name="ls_in_port_sec_l2"}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : "inport == \"provnet-e4abf6df-f8cf-49fd-85d4-3ea399f4d645\""
        pipeline            : ingress
        priority            : 50
        table_id            : 0

        _uuid               : ba45a569-5e0b-4a7e-a939-34bae9f44d34
        actions             : "next;"
        external_ids        : {stage-name=ls_in_port_sec_ip}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : "1"
        pipeline            : ingress
        priority            : 0
        table_id            : 1

        _uuid               : 766938a1-71d3-4be3-b5ce-0f94de1ed303
        actions             : "next;"
        external_ids        : {stage-name=ls_in_port_sec_nd}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : "1"
        pipeline            : ingress
        priority            : 0
        table_id            : 2

        _uuid               : 43cd956f-d536-4a6f-9d2b-d2a4be171039
        actions             : "next;"
        external_ids        : {stage-name=ls_in_pre_acl}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : "1"
        pipeline            : ingress
        priority            : 0
        table_id            : 3

        _uuid               : ad001687-d167-4989-a805-af128f9f26b2
        actions             : "next;"
        external_ids        : {stage-name=ls_in_pre_lb}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : "1"
        pipeline            : ingress
        priority            : 0
        table_id            : 4

        _uuid               : 480e38bf-b5ec-4f26-ab75-2dd0aa352ac2
        actions             : "ct_next;"
        external_ids        : {stage-name=ls_in_pre_stateful}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : "reg0[0] == 1"
        pipeline            : ingress
        priority            : 100
        table_id            : 5

        _uuid               : cfdecdbf-dc46-422a-910b-b3003966c802
        actions             : "next;"
        external_ids        : {stage-name=ls_in_pre_stateful}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : "1"
        pipeline            : ingress
        priority            : 0
        table_id            : 5

        _uuid               : 8e51e8e6-b37a-4d68-afad-80bbee2a87e3
        actions             : "next;"
        external_ids        : {stage-name=ls_in_acl}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : "1"
        pipeline            : ingress
        priority            : 0
        table_id            : 6

        _uuid               : d7968177-6c76-4432-8cb7-e679a7858108
        actions             : "next;"
        external_ids        : {stage-name=ls_in_lb}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : "1"
        pipeline            : ingress
        priority            : 0
        table_id            : 7

        _uuid               : f2232260-ab18-4c91-9b46-e7d39059d478
        actions             : "ct_commit; next;"
        external_ids        : {stage-name=ls_in_stateful}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : "reg0[1] == 1"
        pipeline            : ingress
        priority            : 100
        table_id            : 8

        _uuid               : 1dec4370-6b3a-42a9-83cf-a373636667c9
        actions             : "ct_lb;"
        external_ids        : {stage-name=ls_in_stateful}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : "reg0[2] == 1"
        pipeline            : ingress
        priority            : 100
        table_id            : 8

        _uuid               : 737407d9-4045-4227-accd-869c10fbb7db
        actions             : "next;"
        external_ids        : {stage-name=ls_in_stateful}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : "1"
        pipeline            : ingress
        priority            : 0
        table_id            : 8

        _uuid               : 090f58bd-3da8-41e4-b321-061aeb7eefcb
        actions             : "next;"
        external_ids        : {stage-name=ls_in_arp_rsp}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : "inport == \"provnet-e4abf6df-f8cf-49fd-85d4-3ea399f4d645\""
        pipeline            : ingress
        priority            : 100
        table_id            : 9

        _uuid               : c08246b6-e1b1-4890-a748-ab2c93931c0f
        actions             : "next;"
        external_ids        : {stage-name=ls_in_arp_rsp}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : "1"
        pipeline            : ingress
        priority            : 0
        table_id            : 9

        _uuid               : 72e952e1-9921-46c2-ad35-3fa18241802a
        actions             : "outport = \"_MC_flood\"; output;"
        external_ids        : {stage-name="ls_in_l2_lkup"}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : eth.mcast
        pipeline            : ingress
        priority            : 100
        table_id            : 10

        _uuid               : 1270489f-1937-4e19-80f6-66f4d6b3b86c
        actions             : "outport = \"_MC_unknown\"; output;"
        external_ids        : {stage-name="ls_in_l2_lkup"}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : "1"
        pipeline            : ingress
        priority            : 0
        table_id            : 10

        _uuid               : 6e04cb4b-e86e-4280-b101-a5fb9b436c9a
        actions             : "next;"
        external_ids        : {stage-name=ls_out_pre_lb}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : "1"
        pipeline            : egress
        priority            : 0
        table_id            : 0

        _uuid               : e59532f8-d73e-4087-9b57-758f157cf6ba
        actions             : "next;"
        external_ids        : {stage-name=ls_out_pre_acl}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : "1"
        pipeline            : egress
        priority            : 0
        table_id            : 1

        _uuid               : e3f38136-e766-4764-ba32-e9f19613fe4e
        actions             : "ct_next;"
        external_ids        : {stage-name=ls_out_pre_stateful}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : "reg0[0] == 1"
        pipeline            : egress
        priority            : 100
        table_id            : 2

        _uuid               : 5490b20d-339d-47ef-a02d-6d54275e2a42
        actions             : "next;"
        external_ids        : {stage-name=ls_out_pre_stateful}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : "1"
        pipeline            : egress
        priority            : 0
        table_id            : 2

        _uuid               : 32240c11-7976-4ecc-beb6-f7cabe3b5c32
        actions             : "next;"
        external_ids        : {stage-name=ls_out_lb}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : "1"
        pipeline            : egress
        priority            : 0
        table_id            : 3

        _uuid               : de3786a2-2f2b-4832-98e9-add85adca9d7
        actions             : "next;"
        external_ids        : {stage-name=ls_out_acl}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : "1"
        pipeline            : egress
        priority            : 0
        table_id            : 4

        _uuid               : 41eaf54e-d151-4b6d-95a9-b924dc322ddc
        actions             : "ct_commit; next;"
        external_ids        : {stage-name=ls_out_stateful}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : "reg0[1] == 1"
        pipeline            : egress
        priority            : 100
        table_id            : 5

        _uuid               : 84140145-9d92-4589-952a-a26da694723a
        actions             : "ct_lb;"
        external_ids        : {stage-name=ls_out_stateful}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : "reg0[2] == 1"
        pipeline            : egress
        priority            : 100
        table_id            : 5

        _uuid               : d1d5d9d8-9156-4d2a-a5e0-1f937e861f3a
        actions             : "next;"
        external_ids        : {stage-name=ls_out_stateful}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : "1"
        pipeline            : egress
        priority            : 0
        table_id            : 5

        _uuid               : de2af9be-9298-4040-960e-b5015c842d7c
        actions             : "next;"
        external_ids        : {stage-name=ls_out_port_sec_ip}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : "1"
        pipeline            : egress
        priority            : 0
        table_id            : 6

        _uuid               : 4e6c4521-132b-4b65-9310-10b6fcd9d328
        actions             : "output;"
        external_ids        : {stage-name="ls_out_port_sec_l2"}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : eth.mcast
        pipeline            : egress
        priority            : 100
        table_id            : 7

        _uuid               : 9f4a9f96-a553-40f5-84bf-c84728454555
        actions             : "output;"
        external_ids        : {stage-name="ls_out_port_sec_l2"}
        logical_datapath    : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        match               : "outport == \"provnet-e4abf6df-f8cf-49fd-85d4-3ea399f4d645\""
        pipeline            : egress
        priority            : 50
        table_id            : 7

   * Multicast groups

     .. code-block:: console

        _uuid               : 0102f08d-c658-4d0a-a18a-ec8adcaddf4f
        datapath            : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        name                : _MC_unknown
        ports               : [8427506e-46b5-41e5-a71b-a94a6859e773]
        tunnel_key          : 65534

        _uuid               : fbc38e51-ac71-4c57-a405-e6066e4c101e
        datapath            : f1f0981f-a206-4fac-b3a1-dc2030c9909f
        name                : _MC_flood
        ports               : [8427506e-46b5-41e5-a71b-a94a6859e773]
        tunnel_key          : 65535

Create a subnet on the provider network
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The provider network requires at least one subnet that contains the IP
address allocation available for instances, default gateway IP address,
and metadata such as name resolution.

#. On the controller node, create a subnet bound to the provider network
   ``provider``.

   .. code-block:: console

      $ openstack subnet create --network provider --subnet-range \
        203.0.113.0/24 --allocation-pool start=203.0.113.101,end=203.0.113.250 \
        --dns-nameserver 8.8.8.8,8.8.4.4 --gateway 203.0.113.1 provider-v4
        +-------------------+--------------------------------------+
        | Field             | Value                                |
        +-------------------+--------------------------------------+
        | allocation_pools  | 203.0.113.101-203.0.113.250          |
        | cidr              | 203.0.113.0/24                       |
        | created_at        | 2016-06-15 15:50:45+00:00            |
        | description       |                                      |
        | dns_nameservers   | 8.8.8.8, 8.8.4.4                     |
        | enable_dhcp       | True                                 |
        | gateway_ip        | 203.0.113.1                          |
        | host_routes       |                                      |
        | id                | 32a61337-c5a3-448a-a1e7-c11d6f062c21 |
        | ip_version        | 4                                    |
        | ipv6_address_mode | None                                 |
        | ipv6_ra_mode      | None                                 |
        | name              | provider-v4                          |
        | network_id        | 0243277b-4aa8-46d8-9e10-5c9ad5e01521 |
        | project_id        | b1ebf33664df402693f729090cfab861     |
        | subnetpool_id     | None                                 |
        | updated_at        | 2016-06-15 15:50:45+00:00            |
        +-------------------+--------------------------------------+

If using DHCP to manage instance IP addresses, adding a subnet causes a series
of operations in the Networking service and OVN.

* The Networking service schedules the network on appropriate number of DHCP
  agents. The example environment contains three DHCP agents.

* Each DHCP agent spawns a network namespace with a ``dnsmasq`` process using
  an IP address from the subnet allocation.

* The OVN mechanism driver creates a logical switch port object in the OVN
  northbound database for each ``dnsmasq`` process.

OVN operations
^^^^^^^^^^^^^^

The OVN mechanism driver and OVN perform the following operations
during creation of a subnet on the provider network.

#. If the subnet uses DHCP for IP address management, create logical ports
   ports for each DHCP agent serving the subnet and bind them to the logical
   switch. In this example, the subnet contains two DHCP agents.

   .. code-block:: console

      _uuid               : 5e144ab9-3e08-4910-b936-869bbbf254c8
      addresses           : ["fa:16:3e:57:f9:ca 203.0.113.101"]
      enabled             : true
      external_ids        : {"neutron:port_name"=""}
      name                : "6ab052c2-7b75-4463-b34f-fd3426f61787"
      options             : {}
      parent_name         : []
      port_security       : []
      tag                 : []
      type                : ""
      up                  : true

      _uuid               : 38cf8b52-47c4-4e93-be8d-06bf71f6a7c9
      addresses           : ["fa:16:3e:e0:eb:6d 203.0.113.102"]
      enabled             : true
      external_ids        : {"neutron:port_name"=""}
      name                : "94aee636-2394-48bc-b407-8224ab6bb1ab"
      options             : {}
      parent_name         : []
      port_security       : []
      tag                 : []
      type                : ""
      up                  : true

      _uuid               : 924500c4-8580-4d5f-a7ad-8769f6e58ff5
      acls                : []
      external_ids        : {"neutron:network_name"=provider}
      load_balancer       : []
      name                : "neutron-670efade-7cd0-4d87-8a04-27f366eb8941"
      ports               : [38cf8b52-47c4-4e93-be8d-06bf71f6a7c9,
                             5e144ab9-3e08-4910-b936-869bbbf254c8,
                             a576b812-9c3e-4cfb-9752-5d8500b3adf9]

#. The OVN northbound service creates port bindings for these logical
   ports and adds them to the appropriate multicast group.

   * Port bindings

     .. code-block:: console

        _uuid               : 030024f4-61c3-4807-859b-07727447c427
        chassis             : fc5ab9e7-bc28-40e8-ad52-2949358cc088
        datapath            : bd0ab2b3-4cf4-4289-9529-ef430f6a89e6
        logical_port        : "6ab052c2-7b75-4463-b34f-fd3426f61787"
        mac                 : ["fa:16:3e:57:f9:ca 203.0.113.101"]
        options             : {}
        parent_port         : []
        tag                 : []
        tunnel_key          : 2
        type                : ""

        _uuid               : cc5bcd19-bcae-4e29-8cee-3ec8a8a75d46
        chassis             : 6a9d0619-8818-41e6-abef-2f3d9a597c03
        datapath            : bd0ab2b3-4cf4-4289-9529-ef430f6a89e6
        logical_port        : "94aee636-2394-48bc-b407-8224ab6bb1ab"
        mac                 : ["fa:16:3e:e0:eb:6d 203.0.113.102"]
        options             : {}
        parent_port         : []
        tag                 : []
        tunnel_key          : 3
        type                : ""

   * Multicast groups

     .. code-block:: console

        _uuid               : 39b32ccd-fa49-4046-9527-13318842461e
        datapath            : bd0ab2b3-4cf4-4289-9529-ef430f6a89e6
        name                : _MC_flood
        ports               : [030024f4-61c3-4807-859b-07727447c427,
                               904c3108-234d-41c0-b93c-116b7e352a75,
                               cc5bcd19-bcae-4e29-8cee-3ec8a8a75d46]
        tunnel_key          : 65535

#. The OVN northbound service translates the logical ports into
   additional logical flows in the OVN southbound database.

   .. code-block:: console

      _uuid               : 73c26265-c623-46ac-8fff-8a3dfd7890a6
      actions             : "next;"
      external_ids        : {stage-name="ls_in_port_sec_l2"}
      match               : "inport == \"94aee636-2394-48bc-b407-8224ab6bb1ab\""
      pipeline            : ingress
      priority            : 50
      table_id            : 0

      _uuid               : 50721806-41e4-40e0-bbc6-ffc4d19747c2
      actions             : "next;"
      external_ids        : {stage-name="ls_in_port_sec_l2"}
      logical_datapath    : bd0ab2b3-4cf4-4289-9529-ef430f6a89e6
      match               : "inport == \"6ab052c2-7b75-4463-b34f-fd3426f61787\""
      pipeline            : ingress
      priority            : 50
      table_id            : 0

      _uuid               : 1bf827d4-cf49-4f1f-9218-a62def3d2026
      actions             : "eth.dst = eth.src; eth.src = fa:16:3e:57:f9:ca; arp.op = 2; /* ARP reply \*/ arp.tha = arp.sha; arp.sha = fa:16:3e:57:f9:ca; arp.tpa = arp.spa; arp.spa = 203.0.113.101; outport = inport; inport = \"\"; /* Allow sending out inport. \*/ output;"
      external_ids        : {stage-name=ls_in_arp_rsp}
      logical_datapath    : bd0ab2b3-4cf4-4289-9529-ef430f6a89e6
      match               : "arp.tpa == 203.0.113.101 && arp.op == 1"
      pipeline            : ingress
      priority            : 50
      table_id            : 9

      _uuid               : ae003a1d-42c5-4830-ae7d-62ee70cbd203
      actions             : "eth.dst = eth.src; eth.src = fa:16:3e:e0:eb:6d; arp.op = 2; /* ARP reply \*/ arp.tha = arp.sha; arp.sha = fa:16:3e:e0:eb:6d; arp.tpa = arp.spa; arp.spa = 203.0.113.102; outport = inport; inport = \"\"; /* Allow sending out inport. \*/ output;"
      external_ids        : {stage-name=ls_in_arp_rsp}
      logical_datapath    : bd0ab2b3-4cf4-4289-9529-ef430f6a89e6
      match               : "arp.tpa == 203.0.113.102 && arp.op == 1"
      pipeline            : ingress
      priority            : 50
      table_id            : 9

      _uuid               : b5c2112a-042f-477a-9c23-73c6a70d9145
      actions             : "outport = \"6ab052c2-7b75-4463-b34f-fd3426f61787\"; output;"
      external_ids        : {stage-name="ls_in_l2_lkup"}
      logical_datapath    : bd0ab2b3-4cf4-4289-9529-ef430f6a89e6
      match               : "eth.dst == fa:16:3e:57:f9:ca"
      pipeline            : ingress
      priority            : 50
      table_id            : 10

      _uuid               : 30b41b0b-50dc-4beb-b209-0e5dcfc6ca03
      actions             : "outport = \"94aee636-2394-48bc-b407-8224ab6bb1ab\"; output;"
      external_ids        : {stage-name="ls_in_l2_lkup"}
      logical_datapath    : bd0ab2b3-4cf4-4289-9529-ef430f6a89e6
      match               : "eth.dst == fa:16:3e:e0:eb:6d"
      pipeline            : ingress
      priority            : 50
      table_id            : 10

      _uuid               : fbbbf94f-ab4c-4989-b2c4-f19d67b277dd
      actions             : "output;"
      external_ids        : {stage-name="ls_out_port_sec_l2"}
      logical_datapath    : bd0ab2b3-4cf4-4289-9529-ef430f6a89e6
      match               : "outport == \"6ab052c2-7b75-4463-b34f-fd3426f61787\""
      pipeline            : egress
      priority            : 50
      table_id            : 7

      _uuid               : 6de3e6cf-bfb6-46e1-88cc-745a0fbe0a6c
      actions             : "output;"
      external_ids        : {stage-name="ls_out_port_sec_l2"}
      logical_datapath    : bd0ab2b3-4cf4-4289-9529-ef430f6a89e6
      match               : "outport == \"94aee636-2394-48bc-b407-8224ab6bb1ab\""
      pipeline            : egress
      priority            : 50
      table_id            : 7

#. For each compute node without a DHCP agent on the subnet:

   * The OVN controller service translates the logical flows into flows on the
     integration bridge ``br-int``.

     .. code-block:: console

        cookie=0x0, duration=22.303s, table=32, n_packets=0, n_bytes=0,
            idle_age=22, priority=100,reg7=0xffff,metadata=0x4
            actions=load:0x4->NXM_NX_TUN_ID[0..23],
                set_field:0xffff/0xffffffff->tun_metadata0,
                move:NXM_NX_REG6[0..14]->NXM_NX_TUN_METADATA0[16..30],
                output:5,output:4,resubmit(,33)

#. For each compute node with a DHCP agent on a subnet:

   * Creation of a DHCP network namespace adds two virtual switch ports.
     The first port connects the DHCP agent with ``dnsmasq`` process to the
     integration bridge and the second port patches the integration bridge
     to the provider bridge ``br-provider``.

     .. code-block:: console

        # ovs-ofctl show br-int
        OFPT_FEATURES_REPLY (xid=0x2): dpid:000022024a1dc045
        n_tables:254, n_buffers:256
        capabilities: FLOW_STATS TABLE_STATS PORT_STATS QUEUE_STATS ARP_MATCH_IP
        actions: output enqueue set_vlan_vid set_vlan_pcp strip_vlan mod_dl_src mod_dl_dst mod_nw_src mod_nw_dst mod_nw_tos mod_tp_src mod_tp_dst
         7(tap6ab052c2-7b): addr:00:00:00:00:10:7f
             config:     PORT_DOWN
             state:      LINK_DOWN
             speed: 0 Mbps now, 0 Mbps max
         8(patch-br-int-to): addr:6a:8c:30:3f:d7:dd
            config:     0
            state:      0
            speed: 0 Mbps now, 0 Mbps max

        # ovs-ofctl -O OpenFlow13 show br-provider
        OFPT_FEATURES_REPLY (OF1.3) (xid=0x2): dpid:0000080027137c4a
        n_tables:254, n_buffers:256
        capabilities: FLOW_STATS TABLE_STATS PORT_STATS GROUP_STATS QUEUE_STATS
        OFPST_PORT_DESC reply (OF1.3) (xid=0x3):
         1(patch-provnet-0): addr:fa:42:c5:3f:d7:6f
             config:     0
             state:      0
             speed: 0 Mbps now, 0 Mbps max

   * The OVN controller service translates these logical flows into flows on
     the integration bridge.

     .. code-block:: console

        cookie=0x0, duration=17.731s, table=0, n_packets=3, n_bytes=258,
            idle_age=16, priority=100,in_port=7
            actions=load:0x2->NXM_NX_REG5[],load:0x4->OXM_OF_METADATA[],
                load:0x2->NXM_NX_REG6[],resubmit(,16)
        cookie=0x0, duration=17.730s, table=0, n_packets=15, n_bytes=954,
            idle_age=2, priority=100,in_port=8,vlan_tci=0x0000/0x1000
            actions=load:0x1->NXM_NX_REG5[],load:0x4->OXM_OF_METADATA[],
                load:0x1->NXM_NX_REG6[],resubmit(,16)
        cookie=0x0, duration=17.730s, table=0, n_packets=0, n_bytes=0,
            idle_age=17, priority=100,in_port=8,dl_vlan=0
            actions=strip_vlan,load:0x1->NXM_NX_REG5[],
                load:0x4->OXM_OF_METADATA[],load:0x1->NXM_NX_REG6[],
                resubmit(,16)
        cookie=0x0, duration=17.732s, table=16, n_packets=0, n_bytes=0,
            idle_age=17, priority=100,metadata=0x4,
                dl_src=01:00:00:00:00:00/01:00:00:00:00:00
            actions=drop
        cookie=0x0, duration=17.732s, table=16, n_packets=0, n_bytes=0,
            idle_age=17, priority=100,metadata=0x4,vlan_tci=0x1000/0x1000
            actions=drop
        cookie=0x0, duration=17.732s, table=16, n_packets=3, n_bytes=258,
            idle_age=16, priority=50,reg6=0x2,metadata=0x4 actions=resubmit(,17)
        cookie=0x0, duration=17.732s, table=16, n_packets=0, n_bytes=0,
            idle_age=17, priority=50,reg6=0x3,metadata=0x4 actions=resubmit(,17)
        cookie=0x0, duration=17.732s, table=16, n_packets=15, n_bytes=954,
            idle_age=2, priority=50,reg6=0x1,metadata=0x4 actions=resubmit(,17)
        cookie=0x0, duration=21.714s, table=17, n_packets=18, n_bytes=1212,
            idle_age=6, priority=0,metadata=0x4 actions=resubmit(,18)
        cookie=0x0, duration=21.714s, table=18, n_packets=18, n_bytes=1212,
            idle_age=6, priority=0,metadata=0x4 actions=resubmit(,19)
        cookie=0x0, duration=21.714s, table=19, n_packets=18, n_bytes=1212,
            idle_age=6, priority=0,metadata=0x4 actions=resubmit(,20)
        cookie=0x0, duration=21.714s, table=20, n_packets=18, n_bytes=1212,
            idle_age=6, priority=0,metadata=0x4 actions=resubmit(,21)
        cookie=0x0, duration=21.714s, table=21, n_packets=0, n_bytes=0,
            idle_age=21, priority=100,ip,reg0=0x1/0x1,metadata=0x4
            actions=ct(table=22,zone=NXM_NX_REG5[0..15])
        cookie=0x0, duration=21.714s, table=21, n_packets=0, n_bytes=0,
            idle_age=21, priority=100,ipv6,reg0=0x1/0x1,metadata=0x4
            actions=ct(table=22,zone=NXM_NX_REG5[0..15])
        cookie=0x0, duration=21.714s, table=21, n_packets=18, n_bytes=1212,
            idle_age=6, priority=0,metadata=0x4 actions=resubmit(,22)
        cookie=0x0, duration=21.714s, table=22, n_packets=18, n_bytes=1212,
            idle_age=6, priority=0,metadata=0x4 actions=resubmit(,23)
        cookie=0x0, duration=21.714s, table=23, n_packets=18, n_bytes=1212,
            idle_age=6, priority=0,metadata=0x4 actions=resubmit(,24)
        cookie=0x0, duration=21.714s, table=24, n_packets=0, n_bytes=0,
            idle_age=21, priority=100,ipv6,reg0=0x4/0x4,metadata=0x4
            actions=ct(table=25,zone=NXM_NX_REG5[0..15],nat)
        cookie=0x0, duration=21.714s, table=24, n_packets=0, n_bytes=0,
            idle_age=21, priority=100,ip,reg0=0x4/0x4,metadata=0x4
            actions=ct(table=25,zone=NXM_NX_REG5[0..15],nat)
        cookie=0x0, duration=21.714s, table=24, n_packets=0, n_bytes=0,
            idle_age=21, priority=100,ip,reg0=0x2/0x2,metadata=0x4
            actions=ct(commit,zone=NXM_NX_REG5[0..15]),resubmit(,25)
        cookie=0x0, duration=21.714s, table=24, n_packets=0, n_bytes=0,
            idle_age=21, priority=100,ipv6,reg0=0x2/0x2,metadata=0x4
            actions=ct(commit,zone=NXM_NX_REG5[0..15]),resubmit(,25)
        cookie=0x0, duration=21.714s, table=24, n_packets=18, n_bytes=1212,
            idle_age=6, priority=0,metadata=0x4 actions=resubmit(,25)
        cookie=0x0, duration=21.714s, table=25, n_packets=15, n_bytes=954,
            idle_age=6, priority=100,reg6=0x1,metadata=0x4 actions=resubmit(,26)
        cookie=0x0, duration=21.714s, table=25, n_packets=0, n_bytes=0,
            idle_age=21, priority=50,arp,metadata=0x4,
                arp_tpa=203.0.113.101,arp_op=1
            actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],
                mod_dl_src:fa:16:3e:f9:5d:f3,load:0x2->NXM_OF_ARP_OP[],
                move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],
                load:0xfa163ef95df3->NXM_NX_ARP_SHA[],
                move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],
                load:0xc0a81264->NXM_OF_ARP_SPA[],
                move:NXM_NX_REG6[]->NXM_NX_REG7[],
                load:0->NXM_NX_REG6[],load:0->NXM_OF_IN_PORT[],resubmit(,32)
        cookie=0x0, duration=21.714s, table=25, n_packets=0, n_bytes=0,
            idle_age=21, priority=50,arp,metadata=0x4,
                arp_tpa=203.0.113.102,arp_op=1
            actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],
                mod_dl_src:fa:16:3e:f0:a5:9f,
                load:0x2->NXM_OF_ARP_OP[],
                move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],
                load:0xfa163ef0a59f->NXM_NX_ARP_SHA[],
                move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],
                load:0xc0a81265->NXM_OF_ARP_SPA[],
                move:NXM_NX_REG6[]->NXM_NX_REG7[],
                load:0->NXM_NX_REG6[],load:0->NXM_OF_IN_PORT[],resubmit(,32)
        cookie=0x0, duration=21.714s, table=25, n_packets=3, n_bytes=258,
            idle_age=20, priority=0,metadata=0x4 actions=resubmit(,26)
        cookie=0x0, duration=21.714s, table=26, n_packets=18, n_bytes=1212,
            idle_age=6, priority=100,metadata=0x4,
                dl_dst=01:00:00:00:00:00/01:00:00:00:00:00
            actions=load:0xffff->NXM_NX_REG7[],resubmit(,32)
        cookie=0x0, duration=21.714s, table=26, n_packets=0, n_bytes=0,
            idle_age=21, priority=50,metadata=0x4,dl_dst=fa:16:3e:f0:a5:9f
            actions=load:0x3->NXM_NX_REG7[],resubmit(,32)
        cookie=0x0, duration=21.714s, table=26, n_packets=0, n_bytes=0,
            idle_age=21, priority=50,metadata=0x4,dl_dst=fa:16:3e:f9:5d:f3
            actions=load:0x2->NXM_NX_REG7[],resubmit(,32)
        cookie=0x0, duration=21.714s, table=26, n_packets=0, n_bytes=0,
            idle_age=21, priority=0,metadata=0x4
            actions=load:0xfffe->NXM_NX_REG7[],resubmit(,32)
        cookie=0x0, duration=17.731s, table=33, n_packets=0, n_bytes=0,
            idle_age=17, priority=100,reg7=0x2,metadata=0x4
            actions=load:0x2->NXM_NX_REG5[],resubmit(,34)
        cookie=0x0, duration=118.126s, table=33, n_packets=0, n_bytes=0,
            idle_age=118, hard_age=17, priority=100,reg7=0xfffe,metadata=0x4
            actions=load:0x1->NXM_NX_REG5[],load:0x1->NXM_NX_REG7[],
                resubmit(,34),load:0xfffe->NXM_NX_REG7[]
        cookie=0x0, duration=118.126s, table=33, n_packets=18, n_bytes=1212,
            idle_age=2, hard_age=17, priority=100,reg7=0xffff,metadata=0x4
            actions=load:0x2->NXM_NX_REG5[],load:0x2->NXM_NX_REG7[],
                resubmit(,34),load:0x1->NXM_NX_REG5[],load:0x1->NXM_NX_REG7[],
                resubmit(,34),load:0xffff->NXM_NX_REG7[]
        cookie=0x0, duration=17.730s, table=33, n_packets=0, n_bytes=0,
            idle_age=17, priority=100,reg7=0x1,metadata=0x4
            actions=load:0x1->NXM_NX_REG5[],resubmit(,34)
        cookie=0x0, duration=17.697s, table=33, n_packets=0, n_bytes=0,
            idle_age=17, priority=100,reg7=0x3,metadata=0x4
            actions=load:0x1->NXM_NX_REG7[],resubmit(,33)
        cookie=0x0, duration=17.731s, table=34, n_packets=3, n_bytes=258,
            idle_age=16, priority=100,reg6=0x2,reg7=0x2,metadata=0x4
            actions=drop
        cookie=0x0, duration=17.730s, table=34, n_packets=15, n_bytes=954,
            idle_age=2, priority=100,reg6=0x1,reg7=0x1,metadata=0x4
            actions=drop
        cookie=0x0, duration=21.714s, table=48, n_packets=18, n_bytes=1212,
            idle_age=6, priority=0,metadata=0x4 actions=resubmit(,49)
        cookie=0x0, duration=21.714s, table=49, n_packets=18, n_bytes=1212,
            idle_age=6, priority=0,metadata=0x4 actions=resubmit(,50)
        cookie=0x0, duration=21.714s, table=50, n_packets=0, n_bytes=0,
            idle_age=21, priority=100,ip,reg0=0x1/0x1,metadata=0x4
            actions=ct(table=51,zone=NXM_NX_REG5[0..15])
        cookie=0x0, duration=21.714s, table=50, n_packets=0, n_bytes=0,
            idle_age=21, priority=100,ipv6,reg0=0x1/0x1,metadata=0x4
            actions=ct(table=51,zone=NXM_NX_REG5[0..15])
        cookie=0x0, duration=21.714s, table=50, n_packets=18, n_bytes=1212,
            idle_age=6, priority=0,metadata=0x4 actions=resubmit(,51)
        cookie=0x0, duration=21.714s, table=51, n_packets=18, n_bytes=1212,
            idle_age=6, priority=0,metadata=0x4 actions=resubmit(,52)
        cookie=0x0, duration=21.714s, table=52, n_packets=18, n_bytes=1212,
            idle_age=6, priority=0,metadata=0x4 actions=resubmit(,53)
        cookie=0x0, duration=21.714s, table=53, n_packets=0, n_bytes=0,
            idle_age=21, priority=100,ip,reg0=0x4/0x4,metadata=0x4
            actions=ct(table=54,zone=NXM_NX_REG5[0..15],nat)
        cookie=0x0, duration=21.714s, table=53, n_packets=0, n_bytes=0,
            idle_age=21, priority=100,ipv6,reg0=0x4/0x4,metadata=0x4
            actions=ct(table=54,zone=NXM_NX_REG5[0..15],nat)
        cookie=0x0, duration=21.714s, table=53, n_packets=0, n_bytes=0,
            idle_age=21, priority=100,ipv6,reg0=0x2/0x2,metadata=0x4
            actions=ct(commit,zone=NXM_NX_REG5[0..15]),resubmit(,54)
        cookie=0x0, duration=21.714s, table=53, n_packets=0, n_bytes=0,
            idle_age=21, priority=100,ip,reg0=0x2/0x2,metadata=0x4
            actions=ct(commit,zone=NXM_NX_REG5[0..15]),resubmit(,54)
        cookie=0x0, duration=21.714s, table=53, n_packets=18, n_bytes=1212,
            idle_age=6, priority=0,metadata=0x4 actions=resubmit(,54)
        cookie=0x0, duration=21.714s, table=54, n_packets=18, n_bytes=1212,
            idle_age=6, priority=0,metadata=0x4 actions=resubmit(,55)
        cookie=0x0, duration=21.714s, table=55, n_packets=18, n_bytes=1212,
            idle_age=6, priority=100,metadata=0x4,
                dl_dst=01:00:00:00:00:00/01:00:00:00:00:00
            actions=resubmit(,64)
        cookie=0x0, duration=21.714s, table=55, n_packets=0, n_bytes=0,
            idle_age=21, priority=50,reg7=0x3,metadata=0x4
            actions=resubmit(,64)
        cookie=0x0, duration=21.714s, table=55, n_packets=0, n_bytes=0,
            idle_age=21, priority=50,reg7=0x2,metadata=0x4
            actions=resubmit(,64)
        cookie=0x0, duration=21.714s, table=55, n_packets=0, n_bytes=0,
            idle_age=21, priority=50,reg7=0x1,metadata=0x4
            actions=resubmit(,64)
        cookie=0x0, duration=21.712s, table=64, n_packets=15, n_bytes=954,
            idle_age=6, priority=100,reg7=0x3,metadata=0x4 actions=output:7
        cookie=0x0, duration=21.711s, table=64, n_packets=3, n_bytes=258,
            idle_age=20, priority=100,reg7=0x1,metadata=0x4 actions=output:8

