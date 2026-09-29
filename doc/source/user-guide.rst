===========================
systemd_networkd User Guide
===========================

This role configures host networking with ``systemd-networkd``. It writes
``.netdev``, ``.link`` and ``.network`` files into ``/etc/systemd/network``
and, optionally, starts and enables the ``systemd-networkd`` service.
The role also configures ``systemd-resolved``, but that part is not covered
here.

Overview
~~~~~~~~

Networking is described with two lists:

``systemd_netdevs``
  Virtual devices to create: bonds, VLANs, bridges, dummies, veth pairs. Each
  item maps directly onto the sections of a ``.netdev`` file, so any option
  from ``systemd.netdev`` can be used.

``systemd_networks``
  Configuration applied to an interface, physical or virtual. Each item
  produces a ``.network`` file and a ``.link`` file.

A device that is created in ``systemd_netdevs`` normally also needs an entry in
``systemd_networks``, otherwise nothing tells ``networkd`` what to do with it.

Nothing is started unless ``systemd_run_networkd`` is set to ``true``. Until
then the role only lays down configuration files.

Common network options
~~~~~~~~~~~~~~~~~~~~~~

The most frequently used keys in a ``systemd_networks`` item:

``interface``
  Required. Name of the interface to match.

``filename``
  Name of the generated file without its extension. When it is
  omitted the role generates one from the list index, which means file names
  shift around if the list ever changes. Being explicit with ``filename`` is
  highly recommended in production environments.

``address``
  IP address for the interface, or the string ``dhcp``. May also be a list to
  assign several addresses.

``netmask`` / ``gateway``
  Netmask and gateway used together with ``address``.

``mtu``
  MTU for the interface.

``bond`` / ``bridge``
  Name of the bond or bridge this interface is enslaved to.

``static_routes`` / ``routing_rules``
  Lists of routes and routing policy rules.

``config_overrides``
  Free-form dictionary merged into the generated ``.network`` file. Use this
  for anything the keys above do not cover.

The full list of options is documented in ``defaults/main.yml``.

Creating a bond
~~~~~~~~~~~~~~~

A bond needs three things: a ``netdev`` for the bond device, a network entry
for each member interface pointing at it, and a network entry for the bond
itself.

.. code-block:: yaml

    systemd_netdevs:
      - NetDev:
          Name: bond0
          Kind: bond
        Bond:
          Mode: 802.3ad
          TransmitHashPolicy: layer3+4
          MIIMonitorSec: 1s
          LACPTransmitRate: fast

    systemd_networks:
      # Member interfaces
      - interface: "eth0"
        filename: "10-general-eth0"
        bond: "bond0"
        mtu: 9000
      - interface: "eth1"
        filename: "11-general-eth1"
        bond: "bond0"
        mtu: 9000

      # The bond itself
      - interface: "bond0"
        filename: "12-general-bond0"
        address: "10.0.0.10"
        netmask: "255.255.255.0"
        gateway: "10.0.0.1"
        mtu: 9000

Member interfaces must not be given an address of their own.

Creating a VLAN
~~~~~~~~~~~~~~~

A VLAN also needs a ``netdev``, and the parent interface has to be told which
VLAN devices belong to it. That link is made with ``config_overrides`` on the
parent, using a set of VLAN names:

.. code-block:: yaml

    systemd_netdevs:
      - NetDev:
          Name: bond0.110
          Kind: vlan
        VLAN:
          Id: 110

    systemd_networks:
      # Parent interface, listing its VLANs
      - interface: "bond0"
        filename: "12-general-bond0"
        mtu: 9000
        vlan: "bond0.110"


      # The VLAN interface
      - interface: "bond0.110"
        filename: "13-general-bond0-110"
        address: "172.29.236.100"
        netmask: "255.255.252.0"
        mtu: 9000

Add more VLANs to the same parent with a list, e.g. ``vlan: ["bond0.110",
"bond0.120"]``.

Putting a VLAN in a bridge
~~~~~~~~~~~~~~~~~~~~~~~~~~

A typical OpenStack-Ansible host puts each VLAN into a bridge and addresses the
bridge rather than the VLAN device:

.. code-block:: yaml

    systemd_netdevs:
      - NetDev:
          Name: bond0.110
          Kind: vlan
        VLAN:
          Id: 110
      - NetDev:
          Name: br-mgmt
          Kind: bridge

    systemd_networks:
      - interface: "bond0"
        filename: "12-general-bond0"
        mtu: 9000
        vlan: "bond0.110"

      - interface: "bond0.110"
        filename: "13-general-bond0-110"
        bridge: "br-mgmt"
        mtu: 9000

      - interface: "br-mgmt"
        filename: "14-general-br-mgmt"
        address: "172.29.236.100"
        netmask: "255.255.252.0"
        mtu: 9000

Static routes
~~~~~~~~~~~~~

.. code-block:: yaml

    systemd_networks:
      - interface: "br-mgmt"
        filename: "14-general-br-mgmt"
        address: "172.29.236.100"
        netmask: "255.255.252.0"
        static_routes:
          - cidr: "10.100.0.0/16"
            gateway: "172.29.236.1"

NetworkManager on RedHat distributions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

NetworkManager and ``systemd-networkd`` compete for the same interfaces. On
RedHat based distributions the role stops, disables and masks the NetworkManager
units whenever ``systemd_run_networkd`` is enabled. Units that are not installed
on the host are skipped.

Set ``systemd_networkd_disable_network_manager`` to ``false`` to leave
NetworkManager alone, or override
``systemd_networkd_network_manager_services`` to change which units are masked.

Cleaning up old files
~~~~~~~~~~~~~~~~~~~~~

Setting ``systemd_interface_cleanup`` to ``true`` removes any existing
``.network`` and ``.netdev`` files matching ``systemd_networkd_prefix`` before
new ones are written. This is useful when interfaces have been removed from the
inventory, but it will delete files this role wrote on an earlier run, so
generated file names need to be stable.
