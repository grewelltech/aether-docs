Verify Network
----------------

This section goes into depth on how SD-Core (which runs *inside* the
Kubernetes cluster) connects to either physical gNBs or an emulated
RAN (both running *outside* the Kubernetes cluster). It also describes
how to run diagnostics to debug potential problems.

For the purpose of this section, we assume you already have a scalable
cluster running (as outlined in the previous section), SD-Core has
been installed on that cluster, and you have a terminal window open on
the Master node in that cluster.

Network Schematic
~~~~~~~~~~~~~~~~~~~~~~~~~~

:numref:`Figure %s <fig-macvlan>` shows a high-level schematic of
Aether's end-to-end User Plane connectivity, where we start by
focusing on the basics: a single Aether node, a single physical gNB,
and just the UPF container running inside SD-Core. The identifiers
shown in gray in the figure (``10.76.28.187``, ``10.76.28.113``,
``ens18``) are taken from our running example of an actual
deployment (meaning your details will be different). All the other
names and addresses are part of a standard Aether configuration.

.. _fig-macvlan:
.. figure:: figures/Slide24.png
    :width: 700px
    :align: center

    The UPF pod running inside the server hosting Aether, in the
    default configuration: Macvlan interfaces ``access`` and ``core``
    on the interface carrying the server's default route.
    Identifiers ``10.76.28.187``, ``10.76.28.113``, ``ens18`` are
    specific to a particular deployment site.

As shown in the figure, the UPF has two sides, each a Macvlan
interface on the server's physical interface (the one carrying the
default route, ``ens18`` in our example). The ``access`` side connects
the UPF downstream to the RAN (this corresponds to 3GPP's N3
interface) on subnet ``192.168.252.0/24``. The ``core`` side connects
the UPF upstream to the Internet (this corresponds to 3GPP's N6
interface) on subnet ``192.168.250.0/24``. The server is the router on
both subnets, at ``192.168.252.1`` and ``192.168.250.1``, on Macvlan
interfaces of its own that are also named ``access`` and ``core``.
This means, for example, that the ``access`` interface *inside* the
UPF (which is assigned address ``192.168.252.3``) is the destination
IP address of GTP-encapsulated user plane packets from the gNB.

These are OnRamp's defaults, described under `UPF Networking`_ below.
The checks in this section assume them. If you set
``core.networks.<name>.gateway`` to a router's address, OnRamp
configures nothing on the server for that network: there is no
server-side ``access`` or ``core`` interface and no server route for
it. Check the router instead, and capture on the network's ``iface``.

Following this basic schematic, it is possible to verify that the UPF
is connected to the network by checking to see that the ``core`` and
``access`` interfaces are properly configured. This can be done using
``ip``, and you should see results similar to the following:

.. code-block::

   $ ip addr show core
   15: core@ens18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
       link/ether 06:f7:7c:65:31:fc brd ff:ff:ff:ff:ff:ff
       inet 192.168.250.1/24 brd 192.168.250.255 scope global core
          valid_lft forever preferred_lft forever
       inet6 fe80::4f7:7cff:fe65:31fc/64 scope link
          valid_lft forever preferred_lft forever

   $ ip addr show access
   14: access@ens18: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
       link/ether 82:ef:d3:bb:d3:74 brd ff:ff:ff:ff:ff:ff
       inet 192.168.252.1/24 brd 192.168.252.255 scope global access
          valid_lft forever preferred_lft forever
       inet6 fe80::80ef:d3ff:febb:d374/64 scope link
          valid_lft forever preferred_lft forever

The above output from ``ip`` shows the two interfaces visible to the
server, but running *outside* the container. ``kubectl`` can be used
to see what's running *inside* the UPF, where ``bessd`` is the name of
the container image that implements the UPF, and ``access`` and
``core`` are the last two interfaces shown below:

.. code-block::

   $ kubectl -n aether-5gc exec -ti upf-0 -c bessd -- ip addr
   1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
       link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
       inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
       inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
   3: eth0@if30: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
       link/ether 8a:e2:64:10:4e:be brd ff:ff:ff:ff:ff:ff link-netnsid 0
       inet 192.168.84.19/32 scope global eth0
       valid_lft forever preferred_lft forever
       inet6 fe80::88e2:64ff:fe10:4ebe/64 scope link
       valid_lft forever preferred_lft forever
   4: access@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
       link/ether 82:b4:ea:00:50:3e brd ff:ff:ff:ff:ff:ff link-netnsid 0
       inet 192.168.252.3/24 brd 192.168.252.255 scope global access
       valid_lft forever preferred_lft forever
       inet6 fe80::80b4:eaff:fe00:503e/64 scope link
       valid_lft forever preferred_lft forever
   5: core@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
       link/ether 4e:ac:69:31:a3:88 brd ff:ff:ff:ff:ff:ff link-netnsid 0
       inet 192.168.250.3/24 brd 192.168.250.255 scope global core
       valid_lft forever preferred_lft forever
       inet6 fe80::4cac:69ff:fe31:a388/64 scope link
       valid_lft forever preferred_lft forever

When packets flowing upstream from the gNB arrive on the server's
physical interface, they need to be forwarded over the ``access``
interface.  This is done by having the following kernel route
installed, which should be the case if your Aether installation was
successful:

.. code-block::

   $ ip ro|grep "dev access"
   192.168.252.0/24 dev access proto kernel scope link src 192.168.252.1

Within the UPF, the correct behavior is to forward packets between the
``access`` and ``core`` interfaces.  Upstream packets arriving on the
``access`` interface have their GTP headers removed and the raw IP
packets are forwarded to the ``core`` interface.  The routes inside
the UPF's ``bessd`` container will look something like this:

.. code-block::

   $ kubectl -n aether-5gc exec -ti upf-0 -c bessd -- ip route
   default via 169.254.1.1 dev eth0
   default via 192.168.250.1 dev core metric 110
   10.76.28.0/24 via 192.168.252.1 dev access
   10.76.28.113 via 169.254.1.1 dev eth0
   169.254.1.1 dev eth0 scope link
   192.168.250.0/24 dev core proto kernel scope link src 192.168.250.3
   192.168.252.0/24 dev access proto kernel scope link src 192.168.252.3

The route to ``10.76.28.0/24`` via ``192.168.252.1`` is the UPF's
route to the gNBs (``core.networks.access.ran_subnet``): downstream
GTP packets for the gNBs go to the server, which forwards them onto
its LAN. It reads ``10.76.28.0/24`` here because ``ran_subnet`` was
left empty, so it was taken from ``ens18``'s own subnet. With the
Quick Start's ``172.20.0.0/16`` it reads ``172.20.0.0/16 via
192.168.252.1``.

The default route via ``192.168.250.1`` directs upstream packets to
the Internet via the ``core`` interface, with a next hop of the
``core`` interface outside the UPF.  These packets then undergo source
NAT in the kernel and are sent to the IP destination in the packet.
This means that the ``192.168.100.0/24`` addresses assigned to UEs are
not visible beyond the Aether server. The return (downstream) packets
undergo reverse NAT and now have a destination IP address of the UE.
They are forwarded by the kernel to the ``core`` interface by these
rules on the server:

.. code-block::

   $ ip ro |grep "dev core"
   192.168.100.0/24 via 192.168.250.3 dev core proto static
   192.168.250.0/24 dev core proto kernel scope link src 192.168.250.1

The first rule above matches packets to the UEs on the
``192.168.100.0/24`` subnet.  The next hop for these packets is the
``core`` IP address inside the UPF.  The second rule says that next
hop address is reachable on the ``core`` interface outside the UPF.
As a result, the downstream packets arrive in the UPF where they are
GTP-encapsulated with the IP address of the gNB.

If you are not finding the ``access`` and ``core`` interfaces outside
the UPF, check how OnRamp created them. When ``systemd-networkd``
manages the physical interface, they come from ``macvlan.conf``
drop-ins next to that interface's networkd file; otherwise the
``aether-5gc-macvlan`` service creates them, and the install says so
("Falling back to direct macvlan interface setup"). Re-running ``make
5gc-router-install`` rebuilds them. To create them by hand (again
using our running example for the physical interface):

.. code-block::

    $ sudo ip link add core link ens18 type macvlan mode bridge
    $ sudo ip addr add 192.168.250.1/24 dev core
    $ sudo ip link set core up
    $ sudo ip link add access link ens18 type macvlan mode bridge
    $ sudo ip addr add 192.168.252.1/24 dev access
    $ sudo ip link set access up

Beyond this basic understanding, there are three other details of
note. First, we have been focusing on the User Plane because Control
Plane connectivity is much simpler: RAN elements (whether they are
physical gNBs or gNBsim) reach the AMF using the server's actual IP
address (``10.76.28.113`` in our running example). Kubernetes is
configured to forward SCTP packets arriving on port ``38412`` to the
AMF container.

Second, the basic end-to-end schematic shown in :numref:`Figure %s
<fig-macvlan>` assumes each gNB is assigned an address on the same L2
network as the Aether cluster (e.g., ``10.76.28.0/24`` in our example
scenario). This works when the gNB is physical or when we want to run
a single gNBsim traffic source, but once we scale up the gNBsim by
co-locating multiple containers on a single server, we need to
introduce another network so each container has a unique IP address
(even though the containers are all hosted on the same node). This more
complex configuration is depicted in :numref:`Figure %s <fig-gnbsim>`,
where ``172.20.0.0/16`` is the IP subnet for the virtual network (also
implemented by a Macvlan interface, named ``gnbaccess``).

.. _fig-gnbsim:
.. figure:: figures/Slide25.png
    :width: 600px
    :align: center

    A server running multiple instances of gNBsim, connected to
    Aether.

For completeness, :numref:`Figure %s <fig-start>` shows the Macvlan
setup for the Quick Start configuration, where both the ``gnbaccess``
interface and gNBsim container run in the same server as the Core (but
with the container managed by Docker, independent of Kubernetes).

.. _fig-start:
.. figure:: figures/Slide27.png
    :width: 275px
    :align: center

    The Quick Start configuration with all components running in a
    single server.

Finally, all of the configurable parameters used throughout this
section are defined in the ``core`` and ``gnbsim`` sections of the
``vars/main.yml`` file. Note that an empty value for
``core.networks.access.ran_subnet`` implies the physical L2 network is
used to connect RAN elements to the core, as is typically the case
when connecting physical gNBs. Many of the other variables are
explained in subsequent sections, but for a summary, see the
:doc:`Quick Reference </onramp/ref>`.

.. code-block::

    core:
        values_file: "deps/5gc/roles/core/templates/sdcore-5g-values.yaml"
        networks:
            access:
                ran_subnet: "172.20.0.0/16"  # gNB network; "" derives it from the access interface
        helm:
            local_charts: false  # set chart_ref to local path name if true
            chart_ref: oci://ghcr.io/omec-project/sd-core
            chart_version: 6.1.3
        upf:
            mode: af_packet                    # Options: af_packet or dpdk
            instances:
                default:
                    ue_ip_pool: "192.168.100.0/24"
        amf:
            ip: "10.76.28.113"

    gnbsim:
        ...
        router:
            data_iface: ens18
            macvlan:
                subnet_prefix: "172.20"

In this example only one interface is named:
``gnbsim.router.data_iface``, the server interface the gNBsim
containers attach to. The ``core`` section does not name one. By
default, OnRamp puts the UPF on the interface that carries the
server's default route (the one ``ip route show default`` reports), so
you need to name an interface there only to use a different one, as
described next.

UPF Networking
~~~~~~~~~~~~~~~~~

The UPF (User Plane Function) carries the UEs' traffic. It has two
network interfaces, called its two *sides*:

* **access** (3GPP N3) faces the radio network. gNBs send each UE's
  traffic to the UPF's access address inside GTP-U tunnels (UDP port
  ``2152``), and the UPF sends downstream traffic back to the gNBs
  from it.

* **core** (3GPP N6) faces the data network: the Internet or your own
  servers. Traffic leaves and arrives here as ordinary IP packets
  carrying the UE's address.

Two keys in ``vars/main.yml`` describe this, and both are optional:

* ``core.networks`` describes the networks the sides plug into: which
  server interface each one is on, its subnet, and which router serves
  it.

* ``core.upf.instances`` lists the UPFs, and for each one the network
  and address of each side and the address pool its UEs get.

The Default
^^^^^^^^^^^

With neither key set, OnRamp builds the configuration shown in
:numref:`Figure %s <fig-macvlan>`:

.. list-table::
   :widths: 20 40 40
   :header-rows: 1

   * -
     - Access side (N3)
     - Core side (N6)
   * - Server interface
     - The interface carrying the server's default route (``ens18``
       in our example)
     - The same interface
   * - Subnet
     - ``192.168.252.0/24``
     - ``192.168.250.0/24``
   * - Router
     - The server, at ``192.168.252.1`` on a Macvlan interface named
       ``access``
     - The server, at ``192.168.250.1`` on a Macvlan interface named
       ``core``
   * - UPF address
     - ``192.168.252.3``
     - ``192.168.250.3``

UEs get addresses from ``192.168.100.0/24``. The server routes that
pool to the UPF's core address and masquerades (NATs) it when it
leaves through the default-route interface, so UEs can reach the
Internet. The UPF sends downstream traffic for the gNBs to the server,
which forwards it onto the server's LAN; a gNB on that LAN reaches the
UPF with a route to ``192.168.252.0/24`` via the server's address.

Written out in full, the default is the following. You never need to
write it; on a server whose default route is on ``ens18``, leaving
both keys out has exactly this effect:

.. code-block::

    core:
      networks:
        access:
          iface: ens18               # the interface carrying the default route
          subnet: 192.168.252.0/24
          gateway: host              # this server is the router, at 192.168.252.1
          ran_subnet: ""             # the gNBs are on ens18's own subnet (the server's LAN)
        core:
          iface: ens18
          subnet: 192.168.250.0/24
          gateway: host              # this server is the router, at 192.168.250.1
      upf:
        instances:
          default:
            access: { network: access, ip: 192.168.252.3 }
            core: { network: core, ip: 192.168.250.3 }
            ue_ip_pool: 192.168.100.0/24

The shipped ``vars/main.yml`` sets only two of these values:
``ran_subnet: "172.20.0.0/16"``, because gNBsim's simulated gNBs sit
on that subnet, and the UE pool. The install prints the resolved UPF
networking before anything is deployed, so you can check what the
defaults decided.

Changing What Differs
^^^^^^^^^^^^^^^^^^^^^

Set only what differs from the default. To keep everything above but
use a different interface, for example ``eno1``, name it for both
networks:

.. code-block::

    core:
      networks:
        access: { iface: eno1 }
        core: { iface: eno1 }

The addresses, the routing and the NAT stay as in the default, now on
``eno1``. Keep the shipped ``ran_subnet: "172.20.0.0/16"`` under
``access`` if you run gNBsim; for gNBs on ``eno1``'s own LAN, leave it
out. If ``eno1`` does not carry the default route, a ``ran_subnet``
left empty is taken from ``eno1``'s own subnet, and UE traffic is
NATed on both ``eno1`` and the default-route interface, leaving by
whichever one the server's routing table picks.

To change one side only, name just that side. For example, suppose the
server has a second NIC, ``eno2``, cabled to a site router at
``10.60.0.1`` that leads to the Internet, and UE traffic should leave
through that router instead of through this server:

.. code-block::

    core:
      networks:
        core:
          iface: eno2
          subnet: 10.60.0.0/24
          gateway: 10.60.0.1

The access side keeps its default. On the core side:

* The UPF's core address is ``10.60.0.3``, on ``eno2``, and its
  default route is via ``10.60.0.1``.

* Because the gateway is an address rather than ``host``, this server
  configures nothing for the core network: no ``core`` interface, no
  route to the UE pool, and no NAT.

* So the router at ``10.60.0.1`` must route the UE pool
  (``192.168.100.0/24``) to ``10.60.0.3``, and NAT it if UEs should
  reach the Internet.

Network Fields
^^^^^^^^^^^^^^

Each entry in ``core.networks`` is keyed by the network's name.
``access`` and ``core`` are the networks the built-in UPF uses; other
names are only needed for additional UPFs. All fields are optional:

.. list-table::
   :widths: 15 30 55
   :header-rows: 1

   * - Field
     - Default
     - Description
   * - ``iface``
     - The interface carrying the default route
     - The server interface the network is on.
   * - ``vlan``
     - None
     - An 802.1Q VLAN ID. The UPF then attaches to ``<iface>.<vlan>``,
       which ``5gc-router-install`` creates on the server.
   * - ``subnet``
     - ``192.168.252.0/24`` for ``access``, ``192.168.250.0/24`` for
       ``core``
     - The network's subnet. Required for any other network.
   * - ``gateway``
     - ``host``
     - The network's router. ``host`` makes the server the router, at
       the subnet's first address: it creates a Macvlan interface
       named after the network, forwards between it and ``iface``,
       routes each UPF's UE pool to that UPF, and NATs the pools on the
       way out. An IP address names an external router instead, and
       OnRamp configures nothing on the server for the network.
   * - ``attach``
     - ``macvlan``
     - How the UPF attaches to ``iface`` in ``af_packet`` mode.
       ``macvlan`` shares the interface with the server;
       ``host-device`` moves the whole interface into the UPF pod, so
       the server loses it while the UPF runs and the gateway must be
       an external router.
   * - ``resource_name``
     - ``intel.com/intel_sriov_vfio_access`` / ``..._core``
     - The SR-IOV resource pool in ``dpdk`` mode. Required for any
       other network.
   * - ``ran_subnet``
     - The subnet of ``iface``'s own address
     - Access networks only: where the gNBs are. The UPF sends
       downstream traffic for this subnet to the network's gateway.
       Set it when the gNBs are not on ``iface``'s own subnet
       (gNBsim's ``172.20.0.0/16``, or a routed RAN site), or when
       ``attach: host-device`` takes the interface off the server.

UPF Instances
^^^^^^^^^^^^^

Each entry in ``core.upf.instances`` is one UPF, keyed by its name.

* ``default`` is the UPF installed with the Core. It always exists,
  whether or not it is listed.

* Any other name is an additional UPF. ``make aether-add-upfs``
  installs each one as its own ``bess-upf`` Helm release, in namespace
  ``aether-upf-<name>``; see the :doc:`Multiple UPFs
  </onramp/blueprints>` blueprint.

An instance has these fields:

.. list-table::
   :widths: 20 35 45
   :header-rows: 1

   * - Field
     - Default
     - Description
   * - ``access.network``
     - ``access``
     - The network (a name in ``core.networks``) the UPF's access side
       is on.
   * - ``access.ip``
     - The third address of that network's subnet (``.3``)
     - The UPF's access address.
   * - ``core.network``
     - ``core``
     - The network the UPF's core side is on.
   * - ``core.ip``
     - The third address of that network's subnet
     - The UPF's core address.
   * - ``ue_ip_pool``
     - ``192.168.100.0/24`` for ``default``; required for any other UPF
     - The addresses this UPF's UEs get. Pools must not overlap.
   * - ``mode``
     - ``core.upf.mode``
     - ``af_packet`` or ``dpdk``.

A second UPF on networks of its own: its access side is on ``eno3``,
with this server as the router; its core side is on ``eno4``, behind a
router at ``10.80.0.1``, which must route ``192.168.102.0/24`` to
``10.80.0.5``:

.. code-block::

    core:
      networks:
        n3b: { iface: eno3, subnet: 10.70.0.0/24 }
        n6b: { iface: eno4, subnet: 10.80.0.0/24, gateway: 10.80.0.1 }
      upf:
        instances:
          edge-2:
            access: { network: n3b, ip: 10.70.0.5 }
            core: { network: n6b, ip: 10.80.0.5 }
            ue_ip_pool: 192.168.102.0/24

What the Install Refuses
^^^^^^^^^^^^^^^^^^^^^^^^

The install stops before deploying a configuration that cannot work:
a gateway outside its subnet, ``attach: host-device`` with ``gateway:
host``, moving the interface that carries the server's default route
into a pod, a network used by both a ``dpdk`` and an ``af_packet``
UPF, two UPFs sharing an address, overlapping UE pools, unknown keys,
and the keys this section replaced (the message names each one's
replacement). In ``dpdk`` mode both sides of a UPF must be on the same
interface, and ``attach`` does not apply.

Uninstalling the router step restores the host to the state it was in
before the install (forwarding, GRO, and ``sysctl`` settings, plus any
VLAN interfaces OnRamp created).

Packet Traces
~~~~~~~~~~~~~~~~~

Successfully connecting a UE to the Internet involves configuring the
UE, gNB, and SD-Core software in a consistent way; establishing
SCTP-based control plane (N2) and GTP-based user plane (N3)
connections between the base station and Mobile Core; and traversing
multiple IP subnets along the end-to-end path.

Packet traces are the best way to diagnose your deployment, and the
most helpful traces you can capture are shown in the following
commands. You can run these on the Aether server, where we use our
example ``ens18`` interface for illustrative purposes:

.. code-block::

   $ sudo tcpdump -i any sctp -w sctp-test.pcap
   $ sudo tcpdump -i ens18 port 2152 -w n3-outside.pcap
   $ sudo tcpdump -i access port 2152 -w n3-inside.pcap
   $ sudo tcpdump -i core net 192.168.100.0/24 -w n6-inside.pcap
   $ sudo tcpdump -i ens18 net 192.168.100.0/24 -w n6-outside.pcap

The first trace, saved in file ``sctp.pcap``, captures SCTP packets
sent to establish the control path between the base station and the
Mobile Core (over the **N2** interface). Toggling "Mobile Data" on a
physical UE, for example by turning Airplane Mode off and on, will
generate the relevant control plane traffic; gNBsim automatically
triggers this activity.

The second and third traces, saved in files ``n3-outside.pcap`` and
``n3-inside.pcap``, respectively, capture GTP packets (tunneled
through port ``2152`` ) on the RAN side of the UPF. Setting the
interface to ``ens18`` corresponds to "outside" the UPF and setting
the interface to ``access`` corresponds to "inside" the UPF.  Running
``ping`` from a physical UE will generate the relevant user plane traffic
(over the **N3** interface); gNBsim automatically triggers this activity.

Similarly, the fourth and fifth traces, saved in files
``n6-inside.pcap`` and ``n6-outside.pcap``, respectively, capture IP
packets on the Internet side of the UPF (over the **N6** interface).
In these two tests, ``net 192.168.100.0/24`` corresponds to the IP
addresses assigned to UEs by the SMF. Running ``ping`` from a physical
UE will generate the relevant user plane traffic; gNBsim automatically
triggers this activity.

The ``access`` and ``core`` captures apply when the server is the
gateway for those networks (the default). For a network whose
``gateway`` is a router's address, capture on the network's ``iface``
instead.

If the ``n3-outside.pcap`` has packets and the ``n3-inside.pcap`` is
empty (no packets captured), check that the server forwards between
``ens18`` and ``access``. OnRamp installs these rules through the
``5gc-iptables`` service, so start with:

.. code-block::

   $ systemctl status 5gc-iptables
   $ sudo iptables -S FORWARD | grep access

If the rules are missing, re-run ``make 5gc-router-install``. As a
last resort you can add them by hand:

.. code-block::

   $ sudo iptables -A FORWARD -i ens18 -o access -j ACCEPT
   $ sudo iptables -A FORWARD -i access -o ens18 -j ACCEPT
