---
title: A Model for IPv6 Operation in OpenStack
# abbrev:
docname: draft-baker-openstack-ipv6-model-03
date: 2015-04-23

ipr: trust200902
# area:
# wg:
kw: Internet-Draft
cat: informational

coding: us-ascii
pi:
  toc: yes
  sortrefs: yes
  symrefs: yes

author:
	-
	ins: F. Baker
	name: Fred Baker
	role: editor
	org: Cisco Systems
	city: Santa Barbara
	region: California
	code: 93117
	country: USA
	email: fred@cisco.com

	-
	ins: C. Marino
	name: Chris Marino
	org: Cisco Systems
	city: San Jose
	region: California
	code: 95134
	country: USA
	email: chrmarin@cisco.com
	
	-
	ins: I. Wells
	name: Ian Wells
	org: Cisco Systems
	city: San Jose
	region: California
	code: 95134
	email: iawells@cisco.com

	-
	ins: R. Agarwalla
	name: Rohit Agarwalla
	org: Cisco Systems
	city: San Jose
	region: California
	code: 95134
	email: rogarwa@cisco.com

	-
	ins: S. Jeuk
	name: Sebastian Jeuk
	org: Cisco Systems
	city: San Jose
	region: California
	code: 95134
	country: USA
	email: sjeuk@cisco.com

	-
	ins: G. Salgueiro
	name: Gonzalo Salgueiro
	org: Cisco Systems
	city: Research Triangle Park
	region: NC
	code: 27709
	country: US
	email: gsalguei@cisco.com
	
informative:
  Microsoft-Azure:
    title: [Report: Microsoft Buys 100 Acres of Iowa land for Data Center](http://www.datacenterknowledge.com/archives/2014/08/01/report-microsoft-buys-100-acres-iowa-land-data-center/) 
	author:
	  ins: Y. Sverdlik
	  name: Y. Sverdlik
	  org: Data Center Knowledge
	date: 2014-08  

--- abstract

This is an overview of a network model for OpenStack,
designed to dramatically simplify scalable network deployment
and operations.</t> </abstract>

--- note_Design_Principle

> Perfection is achieved, not when there is nothing more to add, but
> when there is nothing left to take away.
>	Antoine de Saint-Exupery


--- middle

# Introduction {#introduction}

OpenStack, and its issues.

## What is OpenStack? {#projects}

OpenStack is a cloud computing orchestration solution developed using
an open source community process. It consists of a collection of
'projects', each implementing the creation, control, and
administration of tenant resources. There are separate OpenStack
projects for managing compute, storage and network resources.

Neutron is the project that manages OpenStack networking. It exposes a
"northbound" (e.g., application-facing) API to the other OpenStack
projects for programmatic control over tenant network
connectivity. The "southbound" (driver) interface is implemented as
one or more device driver plugins that are built to interact with
specific devices in the network. This approach provides the
flexibility to deploy OpenStack networking using a range of
alternative techniques.

An OpenStack tenant, in the Kilo and earlier releases, is required to
create what OpenStack identifies as a 'Neutron Network' connecting
their virtual machines. This Network is instantiated via the plugins
as either a layer 2 network, a layer 3 network, or as an overlay
network. The actual implementation is unknown to the tenant. The
technology used to provide these networks is selected by the OpenStack
operator based upon the requirements of the cloud deployment.

The tenant also is required, in the Kilo and earlier releases, to
specify an 'IP Subnet' for each Network. This specification is made by
providing a CIDR prefix for IPv4 address allocation via DHCP or for
IPv6 address allocation via DHCP or SLAAC. This address range may be
from within the address range of the data center (non-overlapping), or
overlapping <xref target="RFC1918"/> addresses. Tenants may create
multiple Networks, each with its own Subnet.

An OpenStack Subnet is a logical layer 2 network and requires layer 3
routing for packets to exit the Subnet. This is achieved by attaching
the Subnet to a Neutron Router. The Neutron router implements Network
Address Translation for external traffic from tenant networks as well
for providing connectivity to tenant networks from the outside. Using
Linux utilities, OpenStack can support overlapping RFC 1918 addresses
between tenants.

OpenStack Subnets are typically implemented as VLANs in a data
center. When tenant scalability requirements grow large, an overlay
approach is typically used. Because of the difficulties in scaling and
administering large layer 2 and/or overlay networks, some OpenStack
integrations chose not to provide isolated Subnets and simply offer
tenants a layer 3 based network alternative.

OpenStack uses Layer 3 and Layer 2 Linux utilities on hosts to provide
protection against IP/MAC spoofing and ARP poisoning.

## OpenStack Scaling Issues {#openstack-issues}


One of the fundamental requirements of OpenStack Networking (Neutron)
is to provide scalable, isolated tenant networks. Today this is
achieved via L2 segmentation using either a) standard 802.1Q VLANs or
b) an overlay approach based on one of several L2 over L3
encapsulation techniques available today such as 802.1ad, VXLAN, STT
or NVGRE.

However, these approaches still struggle to provide scalable,
transparent, manageable, high performance, isolated tenant networks.
VLAN's don't scale beyond 4096 (2^12) networks and have complex
trunking requirements when tenants span host and racks. IEEE 802.1ad
(QinQ) partially solves that, but adds another limit - at most 2^12
tenants, each of which may have 2^12 VLANs. IP Encapsulation
introduces additional complexity on host computers running hypervisors
as well as impact performance of tenant applications running on
virtual machines. Overlay based isolation techniques may also impair
traditional network monitoring and performance management tools.
Moreover, when these isolated (L2) networks require external access to
other networks or the public Internet, they require even more complex
solutions to accommodate overlapping IP prefixes and network address
translation (NAT).

As more capabilities are built on to these layer 2 based *virtual*
networks, complexity continues to grow.

This draft presents a new Layer 3 based approach to OpenStack
networking using IPv6 that can be deployed natively on IPv6 networks.
It will be shown that this approach can provide tenant isolation
without the limitations of existing alternatives, as well as deliver
high performance networks transparently using a simplified tenant
network connectivity model, without the overhead of encapsulation or
managing overlapping IP addresses and address translations. We note
that some large content providers, notably Google and <xref
target="FaceBook-IPv6">Facebook</xref>, are going in exactly this
direction.
     
# Requirements {#require}

In this section, we attempt to list critical requirements.

## Design approach {#approach}

As a design approach, we presume an IPv6-only data center in a world
that might have IPv4 or IPv6 clients outside of it. This design
explicitly does not depend on VLANs, QinQ, VXLAN, MPLS, Segment
Routing, LISP, IP/IP or GRE tunnels, or any other supporting
encapsulation. Data center operators remain free to use any of those
tools, but they are not required. If we can do everything required for
OpenStack networking with IPv6 alone, these other networking
technologies may be used as optimizations. If we are unable to satisfy
the OpenStack requirements that also is something we wish to know and
understand.

OpenStack is designed to be used by many cloud users or 'tenants'.
Scalable, secure and isolated tenant networks are a requirement for
building a multi-tenant cloud data center. The OpenStack
administrator/operator can design and configure a cloud environment to
provide network isolation using the approach described in this
document, alone, or in combination with any of the above network
technologies . However, all the details of the underlying technology
and implementation details are completely transparent to the tenant
itself.

## Multiple Data Centers {#multicenter}


A common requirement in network and data center operations is
reliability, serviceability, and maintainability of their operations
in the presence of an outage. At minimum, this implies multihoming in
the sense of having multiple upstream ISPs; in many cases, it also
implies multiple and at times duplicate data centers, and tenants
stretched or able to be readily moved or recreated across multiple
data centers.

## Large Data Centers {#large}

Microsoft Azure {{Microsoft-Azure}} has purchased a 100 acre piece of
land for the construction of a single data center.  In terms of
physical space, that is enough for a data center with about half a
million 19' RETMA racks.

With even modest virtual machine density, infrastructure at this scale
easily exhausts the 16M available RFC 1918 private addresses
(i.e. 10.0.0.0/8) and explains the recent efforts by webscale cloud
providers to deploy IPv6 throughout their new datacenters.

## Multi-tenancy {#multitenant}

While it is possible that a single tenant would require a 100 acre
data center, it would be unusual. In most such data centers, one would
expect a large number of tenants.

## Isolation {#isolation}


Isolation is required between tenants, and at times between tenants
hierarchically related to larger tenants.

### Inter-tenant isolation {#interisolation}

A *tenant* is defined as a set of resources under common
administrative control. It may be appropriate for tenants to
communicate with each other within the context of an application or
relationships among their owners or operators. However, unless
specified otherwise, tenants are intended to operate as if they were
on their own company's premises and be isolated from one another.

### Intra-tenant isolation {#intraisolation}

There are often security compartments within a corporate network, just
as there are security barriers between companies. As a result, there
is a recursive isolation requirement: it must be possible to isolate
an identified part of a tenant (which we also think of as a tenant)
from another part of the same tenant.

## Operational Simplicity {#requirement5}

To the extent possible (and, for operators, the concept will bring a
smile), operation of a multi-location multi-tenant data center, and
the design of an application that runs in one, should be simple and
uncoupled.

As discussed in <xref target="RFC3439"/>, this requires that the
operational model required to support a tenant with only two physical
machines, or virtual machines in the same physical chassis, should be
the same as that required to support a tenant running a million
machines in a federated multiple data center application.
Additionally, this same operational model should scale from running a
single tenant up to many thousands of tenants.


      <section anchor="requirement8" title="Address space ">
        As described in <xref target="projects"/>, currently, an OpenStack
        tenant is required to specify a Subnet's CIDR prefix for IP address
        allocation. With this proposal, this is no longer required.
      </section>

      <section anchor="requirement9" title="Data Center Federation">
        It must be possible to extend the architecture across multiple data
        centers. These data centers may be operated by distinct entities, with
        security policies that apply to their interconnection.
      </section>

      <section anchor="pmtu" title="Path MTU Issues">
        An issue in virtualized data center architectures is <xref
        target="RFC1981">Path MTU Discovery</xref> implementation.
        Implementing Path MTU requires the <xref
        target="RFC4443">ICMPv6</xref> Packet Too Big message to get from the
        originating router or middleware to the indicated host, which is in
        this case virtual and potentially hidden within a tunnel. This is a
        special case of the issues raised in <xref target="RFC2923"/>.
      </section>
    </section>

    <section anchor="models" title="Models">
      <t/>

      <section anchor="configuration" title="Configuration Model">
        In the OpenStack model, the cloud computing user, or tenant, is
        building something Edward Yourdon might call a 'structured design' for
        the application they are building. In the 1960's, when Yourdon started
        specifying process and data flow diagrams, these were job steps in a
        deck of Job Control Language cards; in OpenStack, they are multiple,
        individual machines, virtual or physical, running parts of a
        structured application.

        In these, one might find a load balancer that receives and
        distributes requests to request processors, a set of stored data
        processing applications, and the storage they depend on. What is
        important to the OpenStack tenant is that 'this' and 'that'
        communicate, potentially using or not using multicast communications,
        and don't communicate with 'the other'. Typically unnecessary is any
        and all information regarding how this communication actually needs to
        occur (i.e., placement of routers, switches, and IP subnets, prefixes,
        etc.).

        An IPv6 based networking model simplifies the configuration of
        tenant connectivity requirements. Global reachability eliminates the
        need for network address translation devices as well as
        tenant-specified Subnet prefixes (<xref target="requirement8"/>),
        although tenant-specified ULA prefixes or prefixes from the owner of
        the tenant's address space are usable with it. With the exception of
        network security functions, no network devices need to be specified or
        configured to provide connectivity.
      </section>

      <section anchor="addressing" title="Data Center Model">
        The premises of the routing and addressing models are that <list
            style="symbols">
            The address tells the routing system what topological location
            to deliver a packet to, and within that, what interface to deliver
            it to, and

            The routing system should deliver traffic to a resource if and
            only if the sender is authorized to communicate with that
            resource.

            Contrary to the OpenStack Neutron Networking Model, tunnels are
            not necessary to provide tenant network isolation; we include
            resources in a tenant network by a Role-based Access Control
            model, but address the tenant resources within the data center in
            a manner that scales for the data center.
          </list>

        We expect to find the data center to be composed of some minimal
        unit of connectivity and maintenance, such as a rack or row, and
        equipped with one or more Top-of-Rack or End-of-Row switch(es); each
        configured with at least one subnet prefix, perhaps one per such
        switch. For the purposes of this note, these will be called Racks and
        Top-of-Rack switches, and when applied to other architectures the
        appropriate translation needs to be imposed.

        <xref target="zynga"/> describes a relatively typical rack design.
        It is a simple fat-tree architecture, with every device in a pair, so
        that any failure has an immediate hot backup. There are other common
        designs, such as those that consider each rack to be in a 'row' and in
        a 'column', with one or more distribution switches in each.

        <?rfc needLines='22'?>

        <figure anchor="zynga" title="Typical Rack Design">
        <artwork align="center"><![CDATA[

~~~~
       Distribution   Switches connecting
       / Layer /      racks in a pod, and
      /       /       connecting pods
     /       /
  +-+-+   +-+-+       Mutual backup TOR
+-+TOR+---+TOR+-+     switches
| +---+   +---+ |
| +-----------+ |
+-+    host   +-+     Each host has two
| +-----------+ |     Ethernet interfaces
+-+    host   +-+     with separate subnets
| +-----------+ |
| .           . |
| .           . |
| .           . |     Design premise: complete
| +-----------+ |     redundancy, with every
+-+    host   +-+     switch and every cable
| +-----------+ |     backed up by a doppelganger
+-+    host   +-+
  +-----------+
~~~~

        <section anchor="tenant" title="Tenant Address Model">
          Tenant resources need to be told, by configuration or naming, the
          addresses of resources they communicate with. This is true
          regardless of their location or relationship to a given tenant. In
          environments with well-known addresses, this becomes complex and
          unscalable. This was learned very early with Internet hostnames; a
          single 'hostfile' was maintained by a central entity and updated
          daily, which quickly became unwieldy. The result was the development
          of the Domain Name System; the level of indirection between names
          and addresses improved scalability. It also facilitated ongoing
          maintenance. If a service needed multiple servers, or a server
          needed to change its address, that was trivially solved by changing
          the DNS Resource Record; every resource that needed the new address
          would obtain it the next time it queried the DNS. It has also
          facilitated the IPv4/IPv6 transition; a resource that has an IPv6
          address is given a AAAA record in addition to, or to replace, its
          IPv4 A record.

          Similarly, today's reliance on NAPT technology frequently limits
          the capabilities of an application. It works reasonably well for a
          client accessing a client/server application when the protocol does
          not carry addressing information. If there is an expectation that
          one resource's private address will be meaningful to a peer, such as
          when an SIP client presents its address in SDP or an HTTP server
          presents an address in a redirection, either the resource needs to
          understand the difference between an 'inside' and an 'outside'
          address and know which is which, or it needs a traversal algorithm
          that changes the addresses. For peer-to-peer applications, this
          ultimately means providing a network design in which those issues
          don't apply.

          IPv6 provides global addresses, enough of them that there is no
          real expectation of running out any time soon, making these issues
          go away. In addition, with the IPv4 address space running out, both
          globally and within today's large datacenters, there aren't
          necessarily addresses available for an IPv4 application to use, even
          as a floating IP address.

          Hence, the model we propose is that a resource in a tenant is
          told the addresses of the other resources with which it
          communicates. They are IPv6 addresses, and the data center takes
          care to ensure that inappropriate communications do not take
          place.

          <section anchor="gua-tenant"
                   title="Use of Global Unicast Addresses by Tenants">
            A unicast address in an IP network identifies a topological
            location, by association with an IP prefix (which might be for a
            subnet or any aggregate of subnets). It also identifies a single
            interface located within that subnet, which may or may not be
            instantiated at the time. We assume that there is a subnet
            associated with a top-of-rack switch or whatever its counterpart
            would be in a given network design, and that the physical and
            virtual machines located in that rack have addresses in that
            subnet. This is the same prefix that is used by the datacenter
            administrator.
          </section>

          <section anchor="ula" title="Unique Local Addresses">
            A common requirement is that tenants have the use of some form
            of private address space. In an IPv6 network, a <xref
            target="RFC4193">Unique Local IPv6 Unicast Address</xref> may be
            used to accomplish this. In this case, however, the addresses will
            need to be explicitly assigned to physical or virtual machines
            used by the tenant, perhaps using DHCP or YANG, where a standard
            IPv6 address could be allocated using SLAAC, DHCPv6, or other
            technologies.

            The value of this is that the distinction between a Global
            Address and a Unique Local Address is a corner case in the data
            center; a ULA will not generally be useful when communicating
            outside the data center, but within the data center it is
            rational. Tenants have no routing information or other awareness
            of the prefix. This is not intended for use behind a NAPT;
            resources that need accessibility to or from resources outside the
            tenant, and especially outside the data center, need global
            addresses.
          </section>

          <section anchor="multicast" title="Multicast Domains">
            Multicast capability is a capability enjoyed by some groups of
            resources, that enables them to send a single message and have it
            delivered to multiple destinations roughly simultaneously. At the
            link layer, this means sending a message once that is received by
            a specified set of recipient resources using hardware
            capabilities. IP multicast can be implemented on a LAN as
            specified in <xref target="RFC4291"/>, and can also cross multiple
            subnets directly, using routing protocols such as <xref
            target="RFC4601">Protocol Independent Multicast</xref> <xref
            target="RFC4602"/> <xref target="RFC4604"/> <xref
            target="RFC4605"/> <xref target="RFC4607"/>. In IPv6, the model
            would be that when a group of resources is created with a
            multicast capability, it is allocated one or more source-specific
            transient group addresses as defined in section 2.7 of that
            RFC.
          </section>

          <section anchor="ipv4" title="IPv4 Interaction Model">
            OpenStack IPv4 Neutron uses &ldquo;floating IPv4
            addresses&rdquo; &ndash; global or public IPv4 addresses and
            Network Address Translation - to enable remote resources to
            connect to tenant private network endpoints. Tenant end points can
            connect out to remote resources through an &ldquo;External Default
            Gateway&rdquo;. Both of these depend on <xref
            target="RFC2391">NAPT (DNAT/SNAT)</xref> to ensure that IPv4 end
            points are able communicate and at the same time ensure tenant
            isolation.

            If IPv6 is deployed in a data center, there are fundamentally
            three ways a tenant can interact with IPv4 peers: <list
                style="symbols">
                it can run existing IPv4 OpenStack technology in parallel
                with the IPv6 deployment, meaning that IPv4 is present in the
                data center as well, or

                it can translate or encapsulate IPv4 to IPv6, leaving the
                data center IPv6-only.
              </list>

            If the intention of the data center operator is to give each
            system in the data center a separate global address, It can
            translate an incoming IPv4 address to an internal IPv6 address,
            and have the application see the client or peer as a remote IPv6
            system. This would be as described in <xref
            target="I-D.ietf-v6ops-siit-dc"/>.

            If the intention of the data center operator is to give each
            system in the data center a separate global address, but the
            application is uses IPv4-only APIs and therefore cannot
            communicate using IPv6, It can translate an incoming IPv4 address
            to an internal IPv6 address, and translate it back in the vSwitch,
            so that the VM is presented with what appears to be an end-to-end
            IPv4 packet. This is described in <xref
            target="I-D.ietf-v6ops-siit-dc"/>.

            In the event that A+P IPv4 address multiplexing is in view, the
            vSwitch can also implement the mapping of Address and Port using
            either <xref target="I-D.ietf-softwire-map"> Encapsulation
            (MAP)</xref> or <xref target="I-D.ietf-softwire-map-t">
            Translation (MAP-T)</xref>.

            Running the standard IPv4 OpenStack model and this IPv6 model
            in parallel is complex, if for no other reason than that there are
            two fundamental models in use, one with various encapsulations
            hiding overlapping address space and one with non-overlapping
            address space.

            To simplify the network, as noted in <xref target="approach"/>,
            we suggest that the data center be internally IPv6-only, and IPv4
            be translated or encapsulated to IPv6 at the data center edge. The
            advantage is that it enables IPv4 access while that remains in
            use, and as IPv6 takes over, it reduces the impact of vestigial
            support for IPv4.

            Both translation models in have IPv4 traffic come to an <xref
            target="RFC6145">translator</xref><xref target="RFC6146"/> having
            a pre-configured translation, resulting in an IPv6 packet
            indistinguishable from the packet the remote resource might have
            sent had it been IPv6-capable, with one exception. The IPv6
            destination address is that of the endpoint (the same address
            advertised in a AAAA record), but the source address is an <xref
            target="RFC6052">IPv4-Embedded IPv6 Address</xref> with the IPv4
            address of the sender embedded in a prefix used by the
            translator.

            Access to external IPv4 resources is provided in the same way:
            an <xref target="RFC6147">DNS64</xref> server is implemented that
            contains AAAA records with an <xref target="RFC6052">IPv4-Embedded
            IPv6 Address</xref> with the IPv4 address of the remote resource
            embedded in a prefix used by the translator.

            This follows the <xref target="RFC6144">Framework for IPv4/IPv6
            Translation</xref>, making the internal IPv4 address a floating IP
            address attached to an internal IPv6 address, and the external
            'dial-out' address indistinguishable from a native IPv6
            address.
          </section>

          <section title="Legacy IPv4 OpenStack">
            The other possible model, applicable to IPv4-only devices, is
            to run a legacy OpenStack environment inside IPv6 tunnels. This
            preserves the data center IPv6-only, and enables IPv4-only
            applications, notably those whose licenses tie them to IPv4
            addresses, to run. However, it adds significant overhead in terms
            of encapsulation size and network management complexity.
          </section>
        </section>

        <section anchor="dc"
                 title="Use of Global Addresses by the Data Center">
          Every rack and physical host requires an IP prefix that is
          reachable by the OpenStack operator. This will normally be a global
          IPv6 unicast address. For scalability purposes, as isolation is
          handled separately, this is normally the same prefix as is used by
          tenants in the rack.
        </section>
      </section>

      <section anchor="security-isolation"
               title="Inter-tenant security services">
        In this model, the a label is used to identify a set of virtual or
        physical systems under common ownership and administration that are
        authorized to communicate freely among themselves - a tenant. Tenants
        are not generally authorized to communicate with each other, but
        interactions between specified tenants may be authorized, and specific
        systems may be authorized to communicate generally.

        The fundamental premise is that the vSwitch can determine whether a
        VM is authorized to send or receive a given message. It does so by
        finding the label in a message being sent or received and comparing it
        to a locally-held authorization policy. This policy would indicate
        that the VM is permitted to send or receive messages containing one of
        a small list of labels. In the case of a label contained in the IID of
        an IPv6 address, it would also need to verify the prefix used in the
        address, as this type of policy would be specific to an IPv6
        prefix.

        A set of possible choices that were considered is to be found in
        <xref target="label-appendix"/>. The key questions are a list of
        considerations, presented in no particular order: <list
            style="symbols">
            In what way does the approach the IPv6 Path MTU?

            How does the address come into being?

            What security implications apply? For example, how hard would
            it be for a VM to spoof the source address or attack a random
            destination?

            What is the service offered? Can, for example, policy be
            applied at <list style="symbols">
                The sender of a datagram?

                The receiver of a datagram?

                An arbitrary point between the sender and receiver?

                At the data center edge, on arriving or departing traffic
                other data centers?

                At the data center edge, on arriving or departing traffic
                random locations?
              </list>

            In what way does the approach the IPv6 Path MTU?
          </list>
      </section>

      <section anchor="ipv6-isolation"
               title="IPv6 Tenant Isolation using the Label">
        Neutron today already implements a form of <xref
        target="RFC2827">Network Ingress Filtering</xref>. It prevents the VM
        from emitting traffic with an unauthorized MAC, IPv4, or IPv6 source
        address.

        In addition to this, in this model Neutron prevents the VM from
        transmitting a network packet with an unauthorized label value. The VM
        MAY be configured with and authorized to use one of a short list of
        authorized label values, as opposed to simply having its choice
        overridden; in that case, Neutron verifies the value and overwrites
        one not in the list.

        When a hypervisor is about to deliver an IPv6 packet to a VM, it
        checks the label value against a list of values that the VM is
        permitted to receive. If it contains an unauthorized value, the
        hypervisor discards the packet rather than deliver it. If the Flow
        Label is in use, Neutron zeros the label prior to delivery.

        The intention is to hide the label value from malware potentially
        found in the VM, and enable the label to be used as a form of first
        and last hop security. This provides basic tenant isolation, if the
        label is assigned as a tenant identifier, and may be used more
        creatively such as to identify a network management application as
        separate from a managed resource.
      </section>

      <section anchor="routing" title="Isolation in Routing">
        This concept has the weakness that if a packet is not dropped at
        its source, it is dropped at its destination. It would be preferable
        for the packet to be dropped in flight, such as at the top-of-rack
        switch or an aggregation router.

        Concepts discussed in <xref
        target="I-D.baker-ipv6-isis-dst-flowlabel-routing">IS-IS LSP
        Extendibility</xref><xref target="RFC5120"/><xref target="RFC5308"/>
        and <xref target="I-D.baker-ipv6-ospf-dst-flowlabel-routing">OSPFv3
        LSA Extendibility</xref> <xref
        target="I-D.ietf-ospf-ospfv3-lsa-extend"/><xref target="RFC5340"/> may
        be used to isolate tenants in the routing of the data center backbone.
        This is not strictly necessary, if <xref target="ipv6-isolation"/> is
        uniformly and correctly implemented. It does, however, present a
        second defense against misconfiguration, as the filter becomes
        ubiquitous in the data center and as scalable as routing.
      </section>
    </section>

    <section anchor="bcp38" title="BCP 38 Ingress Filtering">
      As noted in <xref target="ipv6-isolation"/>, Neutron today implements
      a form of <xref target="RFC2827">Network Ingress Filtering</xref>. It
      prevents the VM from emitting traffic with an unauthorized MAC, IPv4, or
      IPv6 source address.

      In IPv6, this is readily handled when the address or addresses used
      by a VM are selected by the OpenStack operator. It may then configure a
      per-VM filter with the addresses it has chosen, following logic similar
      to the <xref target="I-D.ietf-savi-dhcp">Source Address Validation
      Solution for DHCP</xref> or <xref target="RFC7219">SEND</xref>. This is
      also true of <xref target="RFC4862">IPv6 Stateless Address
      Autoconfiguration (SLAAC)</xref> when the MAC address is known and not
      shared.

      However, when SLAAC is in use and either the MAC address is unknown
      or SLAAC's <xref target="RFC4941">Privacy Extensions </xref><xref
      target="RFC7217"/>, are in use, Neutron will need to implement the
      provisions of <xref target="RFC6620">FCFS SAVI: First-Come, First-Served
      Source Address Validation</xref> in order to learn the addresses that a
      VM is using and include them in the per-VM filter.
    </section>

    <section title="Moving virtual machines">
      This design supports these kinds of required layer 2 networks with
      the additional use of a layer 2 over layer 3 encapsulation and tunneling
      protocol, such as <xref target="RFC7348">VXLAN</xref>. The important
      point here being that these overlays are used to address specific tenant
      network requirements and NOT deployed to remove the scalability
      limitations of OpenStack networking.

      There are at least three ways VM movement can be accomplished: <list
          style="symbols">
          Recreation of the VM

          VLAN Modification

          Live Migration of a Running Virtual Machine
        </list>

      <section anchor="motion-new-vm" title="Recreation of the VM">
        The simplest and most reliable is to <list style="numbers">
            Create a new VM in the new location,

            Add its address to the DNS Resource Record for the name,
            allowing new references to the name to send transactions
            there,

            Remove the old address from the DNS Resource Record (including
            the SIIT translation, if one exists), ending the use of the old VM
            for new transactions,

            Wait for the period of the DNS Resource Record's lifetime
            (including the SIIT translation, if one exists), as it will get
            new requests throughout that interval,

            Wait for the for the old VM to finish any outstanding
            transactions, and then

            Kill the old VM.
          </list>

        This is obviously not movement of an existing VM, but preservation
        of the same number and function of VMs by creation of a new VM and
        killing the old.
      </section>

      <section anchor="vmotion"
               title="Live Migration of a Running Virtual Machine">
        At
        http://blogs.vmware.com/vsphere/2011/02/vmotion-whats-going-on-under-the-covers.html,
        VMWare describes its capability, called vMotion, in the following
        terms: <list style="numbers">
            Shadow VM created on the destination host.

            Copy each memory page from the source to the destination via
            the vMotion network. This is known as preCopy.

            Perform another pass over the VM's memory, copying any pages
            that changed during the last preCopy iteration.

            Continue this iterative memory copying until no changed pages
            (outstanding to be-copied pages) remain or 100 seconds elapse.

            Stun the VM on the source and resume it on the destination.
          </list>

        In a native-address environment, we add three steps:<list
            style="numbers">
            Shadow VM created on the destination host.

            Copy each memory page from the source to the destination via
            the vMotion network. This is known as preCopy.

            Perform another pass over the VM's memory, copying any pages
            that changed during the last preCopy iteration.

            Continue this iterative memory copying until no changed pages
            (outstanding to be-copied pages) remain or 100 seconds elapse.

            Stitch routing for the old address.

            Stun the VM on the source and resume it on the destination.

            Renumber the VM as instructed in <xref target="RFC4192"/>.

            Unstitch routing for the old address.
          </list>

        If the VM is moved within the same subnet (which usually implies
        the same rack), there is no stitching or renumbering apart from
        ensuring that the MAC address moves with the VM. When the VM moves to
        a different subnet, however, we need to restitch routing, at least
        temporarily. This obviously calls for some definitions. <list
            style="hanging">
            <t hangText="Stitching Routing:">The VM is potentially in
            communication with two sets of peers: VMs in the same subnet, and
            VMs in different subnets. <list style="symbols">
                The router in the new subnet is instructed to advertise a
                host route (/128) to the moved VM, and to install a static
                route to the old address with the VM's address in the new
                subnet as its next hop address. Traffic from VMs from other
                subnets will now follow the host route to the VM in its new
                location.

                The router in the old subnet is instructed to direct LAN
                traffic to the VM's MAC Address to its IPv6 forwarding logic.
                Traffic from other VMs in the old subnet will now follow the
                host route to the moved VM.
              </list>

            <t hangText="Renumbering:">This step is optional, but is good
            hygiene if the VM will be there a while. If the VM will reside in
            its new location only temporarily, it can be skipped. <vspace
            blankLines="1"/> Note that every IPv6 address, unlike an IPv4
            address, has a lifetime. At least in theory, when the lifetime
            expires, neighbor relationships with the address must be extended
            or the address removed from the system. The <xref
            target="RFC4861">Neighbor Discovery</xref> process in the subnet
            router will periodically emit a Router Advertisement; the VM will
            gain an IPv6 address in the new subnet at that time if not
            earlier. As described in <xref target="RFC4192"/>, DNS should be
            changed to report the new address instead of the old. The DNS
            lifetime and any ambient sessions using the old address are now
            allowed to expire. That this point, any new sessions will be using
            the new address, and the old is vestigial. <vspace
            blankLines="1"/> Waiting for sessions using the address to expire
            can take an arbitrarily long interval, because the session
            generally has no knowledge of the lifetime of the IPv6
            address.

            <t hangText="Unstitching Routing:">This is the reverse process of
            stitching. If the VM is renumbered, when the old address becomes
            vestigial, the address will be discarded by the VM; if the VM is
            subsequently taken out of service, it has the same effect. At that
            point, the host route is withdrawn, and the MAC address in the old
            subnet router's tables is removed.
          </list>
      </section>
    </section>

    <section anchor="implications" title="OpenStack implications">
      <section anchor="config-implications" title="Configuration implications">
        <list style="numbers">
            Neutron MUST be configured with a pre-determined default label
            value for each tenant virtual network <xref
            target="ipv6-isolation"/>.

            Neutron MAY be configured with a set of authorized label values
            for each tenant virtual network <xref
            target="ipv6-isolation"/>.

            A virtual tenant network MAY be configured with a set of
            authorized label values <xref target="ipv6-isolation"/>.

            Neutron MUST be configured with one or more label values per
            virtual tenant network that the network is permitted to receive
            <xref target="ipv6-isolation"/>.
          </list>
      </section>

      <section anchor="vSwitch-implications" title="vSwitch implications">
        On messages transmitted by a virtual machine <list style="hanging">
            <t hangText="Label Correctness:">As described in <xref
            target="security-isolation"/>, ensure that the label in the packet
            is one that the VM is authorized to use. Exactly what label is in
            view is a deferred, and potentially configurable, option. Again
            Depending on configuration, the vSwitch may overwrite whatever
            value is there, or may ratify that the value there is as specified
            in a VM-specific list.

            <t hangText="Source Address Validation:">As described in <xref
            target="bcp38"/>, force the source address to be among those the
            VM is authorized to use. The VM may simultaneously be authorized
            to use several addresses.

            <t hangText="Destination Address Validation:">OpenStack for IPv4
            permits a NAT translation, called a 'floating IP address', to
            enable a VM to communicate outside the domain; without that, it
            cannot. For IPv6, the destination address should be permitted by
            some access list, which may permit all addresses, or addresses
            matching one or more CIDR prefixes such as permitted multicast
            addresses, and the prefix of the data center.
          </list>

        On messages received for delivery to a virtual machine <list
            style="hanging">
            <t hangText="Label Authorization:">As described in <xref
            target="ipv6-isolation"/>, the vSwitch only delivers a packet to a
            VM if the VM is authorized to receive it. The VM may have been
            authorized to receive several such labels.
          </list>

        Each approach in the appendix discusses filtering.
      </section>
    </section>

    <section anchor="IANA" title="IANA Considerations">
      This document does not ask IANA to do anything.
    </section>

    <section anchor="Security" title="Security Considerations">
      In <xref target="isolation"/> and <xref
      target="security-isolation"/>, this specification considers inter-tenant
      and intra-tenant network isolation. It is intended to contribute to the
      security of a network, much like encapsulation in a maze of tunnels or
      VLANs might, but without the complexities and overhead of the management
      of such resources. This does not replace the use of IPSec, SSH, or TLS
      encryption or the use authentication technologies; if these would be
      appropriate in an on-premises corporate data center, they remain
      appropriate in a multi-tenant data center regardless of the isolation
      technology. However, one can think of this as a simple inter-tenant
      firewall based on the concepts of role-based access control; if it can
      be readily determined that a sender is not authorized to communicate
      with a receiver, such a transmission is prevented.
    </section>

    <section anchor="Privacy" title="Privacy Considerations">
      This specification places no personally identifying information in an
      unencrypted part of a packet.
    </section>

    <section anchor="Acknowledgements" title="Acknowledgements">
      This document grew out of a discussion among the authors and
      contributors.
    </section>

    <section anchor="Contributors" title="Contributors">
      <figure anchor="a">
        <artwork align="left"><![CDATA[

Puneet Konghot Cisco Systems San Jose, California 95134 USA Email:
pkonghot@cisco.com

Shannon McFarland Cisco Systems Boulder, Colorado 80301 USA Email:
shmcfarl@cisco.com ]]\></artwork>
</figure>
    </section>
  
</middle>

<back> <!-- references split to informative and normative -->

    <references title="Normative References">
      <?rfc include='reference.RFC.2119' ?>

      <?rfc include='reference.RFC.2460' ?>
    </references>

    <references title="Informative References">
      <reference anchor="UCC">
        <front>
          <title>Towards Cloud, Service and Tenant Classification for Cloud
          Computing</title>

          <author fullname="Sebastian Jeuk" initials="S." surname="Jeuk">
            <organization/>
          </author>

          <author fullname="Jakub Szefer" initials="J." surname="Szefer">
            <organization/>
          </author>

          <author fullname="Shi Zhou" initials="S." surname="Zhou">
            <organization/>
          </author>

          <date month="May" year="2014"/>
        </front>

        <seriesInfo name="IEEE Xplore Digital Library:"
                    value="Cluster, Cloud and Grid Computing (CCGrid), 2014 14th IEEE/ACM International Symposium"/>

        <format target="http://ieeexplore.ieee.org/xpls/icp.jsp?arnumber=6846532"
                type="HTML"/>
      </reference>


      <reference anchor="FaceBook-IPv6">
        <front>
          <title>Facebook Is Close to Having an IPv6-only Data Center
          http://blog.ipspace.net/2014/03/facebook-is-close-to-having-ipv6-only.html</title>

          <author fullname="I. Pepelnjak" initials="I." surname="Pepelnjak">
            <organization>Internetworking perspectives by Ivan
            Pepelnjak</organization>
          </author>

          <date month="March" year="2014"/>
        </front>

        <format target="http://blog.ipspace.net/2014/03/facebook-is-close-to-having-ipv6-only.html"
                type="HTML"/>
      </reference>

      <?rfc include="reference.I-D.ietf-6man-default-iids" ?>

      <?rfc include='reference.I-D.baker-ipv6-isis-dst-flowlabel-routing' ?>

      <?rfc include='reference.I-D.baker-ipv6-ospf-dst-flowlabel-routing' ?>

      <?rfc include="reference.I-D.gont-v6ops-ipv6-ehs-in-real-world" ?>

      <?rfc include='reference.I-D.ietf-ospf-ospfv3-lsa-extend' ?>

      <?rfc include='reference.I-D.ietf-savi-dhcp' ?>

      <?rfc include='reference.I-D.ietf-v6ops-siit-dc' ?>

      <?rfc include='reference.I-D.previdi-6man-segment-routing-header' ?>

      <?rfc include="reference.I-D.ietf-v6ops-siit-dc-2xlat" ?>

      <?rfc include="reference.I-D.ietf-softwire-map" ?>

      <?rfc include="reference.I-D.ietf-softwire-map-t" ?>

      <?rfc include='reference.RFC.1918' ?>

      <?rfc include='reference.RFC.1981' ?>

      <?rfc include='reference.RFC.2205' ?>

      <?rfc include='reference.RFC.2391' ?>

      <?rfc include='reference.RFC.2710' ?>

      <?rfc include='reference.RFC.2827' ?>

      <?rfc include='reference.RFC.2923' ?>

      <?rfc include='reference.RFC.3439' ?>

      <?rfc include='reference.RFC.3697' ?>

      <?rfc include='reference.RFC.4192' ?>

      <?rfc include='reference.RFC.4193' ?>

      <?rfc include='reference.RFC.4291' ?>

      <?rfc include='reference.RFC.4443' ?>

      <?rfc include='reference.RFC.4601' ?>

      <?rfc include='reference.RFC.4602' ?>

      <?rfc include='reference.RFC.4604' ?>

      <?rfc include='reference.RFC.4605' ?>

      <?rfc include='reference.RFC.4607' ?>

      <?rfc include='reference.RFC.4861' ?>

      <?rfc include='reference.RFC.4862' ?>

      <?rfc include='reference.RFC.4941' ?>

      <?rfc include='reference.RFC.5120' ?>

      <?rfc include='reference.RFC.5308' ?>

      <?rfc include='reference.RFC.5340' ?>

      <?rfc include='reference.RFC.5548' ?>

      <?rfc include='reference.RFC.5673' ?>

      <?rfc include='reference.RFC.6052' ?>

      <?rfc include='reference.RFC.6144' ?>

      <?rfc include='reference.RFC.6145' ?>

      <?rfc include='reference.RFC.6146' ?>

      <?rfc include='reference.RFC.6147' ?>

      <?rfc include='reference.RFC.6437' ?>

      <?rfc include='reference.RFC.6620' ?>

      <?rfc include='reference.RFC.7217' ?>

      <?rfc include='reference.RFC.7219' ?>

      <?rfc include='reference.RFC.7348' ?>
    </references>

    <section anchor="log" title="Change Log">
      <list style="hanging">
          <t hangText="Initial Version:">October 2014

          <t hangText="First update:">January 2015
        </list>
    </section>

    <section anchor="label-appendix" title="Alternative Labels considered">
      Several different approaches have been proposed to labeling packets.
      The one we are prototyping first is <xref target="B-label-IID"/>.

      A use case that is primary in our thinking is shown in xref
      target="use-case-1"/&gt;.

      <figure anchor="use-case-1"
              title="Multi-site Multi-administration data center application">
        <artwork align="center"><![CDATA[

                       ,---------.
                     ,'Tenant A's `.
                    (  Home Network )
                     `.           ,'
                       `----+----'
                            |  2001:db8:3::/48
                   ,--------+--------.

Data Center ( "the Internet" ) Data Center 2001:db8:1::/48
`-----+------+----'    2001:db8:2::/48             ,-----.     /        \     ,-----.           ,'`.
/   ,'
`.          /           \/            \/           \         /  ,-------.  \            /  ,-------.  \        / ,'Tenant A`.
  / ,'Tenant A
`. \       / (  0xA12345   ) \        / (  0xC12345   ) \      ;`. ,' :
; `.         ,'   :      |`-------' | |
`-------'     |      |                   |      |                   |      |     ,-------.     |      |     ,-------.     |      :   ,'Tenant B`.
; : ,'Tenant C
`.   ;       \ (  0xB12345   ) /        \ (  0xA12345   ) /        \`.
,' /  `.         ,' /         \`-------' /  
`-------'  /          \           /              \           /`. ,' \`.
,' '-----' '-----'

    ]]></artwork>
      </figure>

      In this use case, there exist two data centers that one might presume
      are operated by different entities. The two data centers use separate
      address spaces, 2001:db8:1::/48 and 2001:db8:2::/48. One tenant (which
      is to say, a set of systems, virtual or physical, that normally
      communicate with each other), Tenant A, is spread across the two data
      centers, and has a different Tenant ID in each of them. One
      administration has an additional tenant, Tenant B, and the other also
      has an additional tenant, Tenant C. For the sake of amusement, we
      presume that Tenant C, in the right data center, has the same tenant ID
      as Tenant A uses in the left data center. Additionally, Tenant A has a
      "home site" that is authorized to communicate with its data center
      systems.

      From the perspective of communication management, therefore, to
      connect Tenant A to itself and deny all other communications, we need
      four filter rules: <list style="symbols">
          Permit messages from/to 2001:db8:1::/48 with tenant ID
          0xA12345,

          Permit messages from/to 2001:db8:2::/48 with tenant ID
          0xC12345,

          Permit messages from/to 2001:db8:3::/48

          Deny everything else
        </list>

      Ideally, one would like to apply those rules to the destination
      address at the sender of a packet, and the source address at the
      receiver of a packet. And one would like to do so in a manner that
      minimizes the possibility of disruptive packet filtering such as
      discussed in <xref target="I-D.gont-v6ops-ipv6-ehs-in-real-world"/>.

      In addition, one wants a BCP-38 filter, ensuring that packets sent by
      a system (virtual or physical) uses only one of the source addresses it
      is supposed to be using.

      <section anchor="B-label-2460" title="IPv6 Flow Label">
        The IPv6 flow label may be used to identify a tenant or part of a
        tenant, and to facilitate access control based on the flow label
        value. The flow label is a flat 20 bits, facilitating the designation
        of 2^20 (1,048,576) tenants without regard to their location.
        1,048,576 is less than infinity, but compared to current data centers
        is large, and much simpler to manage.

        Note that this usage differs from the current <xref
        target="RFC6437">IPv6 Flow Label Specification</xref>. It also differs
        from the use of a flow label recommended by the <xref
        target="RFC2460">IPv6 Specification</xref>, and the respective usages
        of the flow label in the <xref target="RFC2205">Resource ReSerVation
        Protocol</xref> and the previous <xref target="RFC3697">IPv6 Flow
        Label Specification</xref>, and the projected usage in <xref
        target="RFC5548">Low-Power and Lossy Networks</xref><xref
        target="RFC5673"/>. Within a target domain, the usage may be specified
        by the domain. That is the viewpoint taken in this specification.

        <section title="Metaconsiderations">
          <section title="Service offered">
            To Be Supplied
          </section>

          <section title="Pros and Cons">
            <section title="The case in favor of this approach">
              To Be Supplied
            </section>

            <section title="The case against">
              To Be Supplied
            </section>
          </section>

          <section title="Filtering considerations">
            To Be Supplied
          </section>
        </section>
      </section>

      <section anchor="B-label-federated" title="Federated Identity">
        <section title="Introduction">
          In the course of developing draft-baker-ipv6-openstack-model, it
          was determined that a way was needed to encode a federated identity
          for use in Role-Based Access Control. This appendix describes an
          <xref target="RFC2460">IPv6</xref> option that could be carried in
          the Hop-by-Hop or Destination Options Header. The format of an
          option is defined in section 4.2 of that document, and the
          Hop-by-Hop and Destination Options are defined in sections 4.3 and
          4.6 of that document respectively.

          A 'Federated Identity', in the words of the Wikipedia, 'is the
          means of linking an electronic identity and attributes, stored
          across multiple distinct identity management systems.' In this
          context, it is a fairly weak form of that; it is intended for quick
          interpretation in an access list at the Internet layer as opposed to
          deep analysis for login or other security purposes at the
          application layer, and rather than identifying an individual or a
          system, it identifies a set of systems whose members are authorized
          to communicate freely among themselves and may also be authorized to
          communicate with other identified sets of systems. Either two
          systems are authorized to communicate or they are not, and
          unauthorized traffic can be summarily discarded. The identifier is
          defined in a hierarchical fashion, for flexibility and
          scalability.

          'Role-Based Access Control', in this context, applies to groups
          of virtual or physical hosts, not individuals. In the simplest case,
          the several tenants of a multi-tenant data center might be
          identified, and authorized to communicate only with other systems
          within the same 'tenant' or with identified systems in other tenants
          that manage external access. One could imagine a company purchasing
          cloud services from multiple data center operators, and as a result
          wanting to identify the systems in its tenant in one cloud service
          as being authorized to communicate with the systems its tenant of
          the other. One could further imagine a given department within that
          company being authorized to speak only with itself and an identified
          set of other departments within the same company. To that end, when
          a datagram is sent, it is tagged with the federated identify of the
          sender (e.g., {datacenter, client, department}), and the receiving
          system filters traffic it receives to limit itself to a specific set
          of authorized communicants.

          <!--
    <section title='Requirements Language'>
      The key words 'MUST', 'MUST NOT', 'REQUIRED', 'SHALL', 'SHALL NOT',
      'SHOULD', 'SHOULD NOT', 'RECOMMENDED', 'MAY', and 'OPTIONAL' in this
      document are to be interpreted as described in <xref
      target='RFC2119'></xref>.
    </section>
    -->
        </section>

        <section anchor="B-sect2" title="Federated identity Option">
          The option is defined as a sequence of numbers that identify
          relevant parties hierarchically. The specific semantics (as in, what
          number identifies what party) are beyond the scope of this
          specification, but they may be interpreted as being successively
          more specific; as shown in <xref target="B-hierarchical"/>, the
          first might identify a cloud operator, the second, if present, might
          identify a client of that operator, and the third, if present, might
          identify a subset of that client's systems. In an application
          entirely used by Company A, there might be only one number, and it
          would identify sets of systems important to Company A such as
          business units. If Company A uses the services of a multi-tenant
          data center #1, it might require that there be two numbers,
          identifying Company A and its internal structure. If Company A uses
          the services of both multi-tenant data centers #1 and #2, and they
          are federated, the identifier might need to identify the data
          center, the client, and the structure of the client.

          <figure anchor="B-hierarchical"
                  title="Use case: Identifying authorized communicatants in an RBAC environment">
            <artwork align="center"><![CDATA[
                   _.----------------------.
           _.----''                         `------.
      ,--''                                         `---.

,-' DataCenter .---------------------.
`-.  ,'    Company #2,---''      Unauthorized`----.
`. ;              ,' ,-----+-----.        ,--+--------.`. : | ( (
Department 1)--//--( Department 2) ) | ; `.`-----+----+'
`-----------','        |`.
`----. |     X Company A     _.---'        ,'    '-.                 A|------X------------''           ,-'`---.
u| X *.--'
`------.    t|        X          _.----''`---h|---------X-------'' o| X
*.--r+-----------X------. *.----'' i| X
`------.       ,--''            z|             X`---. ,-' DataCenter
e|--------------X-----.
`-.  ,'    Company #2,---''d|               X`----.
`. ;              ,' ,-----+-----.        ,-+---------.`. : | ( (
Department 1) ( Department 2) ) | ; `.`-----------'
`-----------','        |`.
`----.     Company A         _.---'        ,'    '-.`--------------------''
,-' `---.                                         _.--'`------. *.----''
\`----------------------'' ]]\></artwork>
</figure>

          <section anchor="B-format" title="Option Format">
            A number (<xref target="B-number"/>) is represented as a base
            128 number whose coefficients are stored in the lower 7 bits of a
            string of bytes. The upper bit of each byte is zero, except in the
            final byte, in which case it is 1. The most significant
            coefficient of a non-zero number is never zero.

            <figure anchor="B-number" title="Sample numbers">
              <artwork align="center"><![CDATA[

8 = 8\*128\^0 +-+------+ |1| 8 | +-+------+

987 = 7*128\^1 + 91*128\^0 +-+------+-+------+ |0| 7 |1| 91 |
+-+------+-+------+

121393 = 7*128\^2 + 52*128\^1 + 49\*128\^0 +-+------+-+------+-+------+
|0| 7 |0| 52 |1| 49 | +-+------+-+------+-+------+ ]]\></artwork>
</figure>

            The identifier {8, 987, 121393} looks like

            <figure anchor="B-identifier" title="">
              <artwork align="center"><![CDATA[

+-------+-------+-+-----+-+-----+-+-----+-+-----+-+-----+-+-----+ | type
| len=6 |1| 8 |0| 7 |1| 91 |0| 7 |0| 52 |1| 49 |
+-------+-------+-+-----+-+-----+-+-----+-+-----+-+-----+-+-----+
]]\></artwork>
</figure>

            <section anchor="B-destopt"
                     title="Use in the Destination Options Header">
              In an environment in which the validation of the option only
              occurs in the receiving system or its hypervisor, this option is
              best placed in the Destination Options Header.
            </section>

            <section anchor="B-hopbyhop" title="Use in the Hop-by-Hop Header">
              In an environment in which the validation of the option
              occurs in transit, such as in a firewall or other router, this
              option is best placed in the Hop-by-Hop Header.
            </section>
          </section>
        </section>

        <section title="Metaconsiderations">
          <section title="Service offered">
            To Be Supplied
          </section>

          <section title="Pros and Cons">
            <section title="The case in favor of this approach">
              To Be Supplied
            </section>

            <section title="The case against">
              To Be Supplied
            </section>
          </section>

          <section title="Filtering considerations">
            To Be Supplied
          </section>
        </section>
      </section>

      <section anchor="Appendix_UCC" title="Universal Cloud Classification">
        <section anchor="UCC_Intro" title="Introduction">
          Cloud environments suffer from ambiguity in identifying their
          services and tenants. Traffic from different cloud providers cannot
          be distinguished easily on the Internet. Filters are simply not able
          to obtain the provider, service and tenant identities from network
          packets without leveraging other more latency intense inspection
          methods. This appendix describes the Universal Cloud Classification
          (UCC) <xref target="UCC"/> approach as a way to identify cloud
          providers, their services and tenants on the network layer. It
          introduces a Cloud-ID, Service-ID and Tenant-ID. The IDs are
          incorporated into an IPv6 extension header and can be used for
          different use-cases both within and outside a Cloud Environment. The
          format of the IDs and their characteristics are defined in <xref
          target="UCC_Options"/> of the document and the extension header is
          defined in <xref target="UCC_Extensions_Header"/>.

          Applications and users are defined in many different ways in
          cloud environments, therefore ambiguity is multifold:<list
              style="numbers">
              The first ambiguity is described by how a service is defined
              in cloud environments. Here, an application within a Cloud
              Provider is called a service. However, the cloud providers
              network can not distinguish services from services run on top of
              other services. Distinguishing sub-services hosted by a service
              becomes critical when applying network services to specific
              sub-services.

              Secondly, a tenant in a cloud provider can have different
              meanings. Here, tenant is used to define a consumer of a cloud
              service. At the same time a service run on top of another
              service can be considered a tenant of that particular service.
              These ambiguities make it extremely difficult to uniquely
              identify services and their tenants in cloud environments. This
              multi-layered service and tenant relationship is one of the most
              complex tasks to handle using existing technologies.
            </list>

          A service can be defined as a group of entities offering a
          specific function to a tenant within a Cloud Provider.
        </section>

        <section anchor="UCC_Options"
                 title="Universal Cloud Classification Options">
          Three IDs are defined that classify a Tenant specific to the
          service used within a certain Cloud Provider. The Cloud-ID,
          Service-ID and Tenant-ID are defined hierarchically and support
          service-stacking. The IDs are based on the 'Digital Object
          Identifier' scheme and support incorporating metadata per ID. The ID
          can be of variable length but Cloud-ID, Service-ID and Tenant-ID are
          proposed within a 4 byte, 6 byte and 6 byte tuple respectively.

          <section anchor="UCC_Cloud_ID" title="Cloud ID">
            The Cloud ID is a globally unique ID that is managed by a
            registrar similar to DNS. It is 4 bytes in sizes and defined with
            a 10 bit location part and a 22 bit provider ID.
          </section>

          <section anchor="UCC_Service_ID" title="Service ID">
            The Service ID is used to identify a service both within and
            outside a cloud environment. It is a 6 byte long ID that is
            separated into several sub-IDs defining the data center, service
            and an option field. The Data Center location is defined by 8
            bits, the Service is 32 bits long and the Option field provides
            another 8 bits. The option bits can be used to incorporate
            information used for en-route or destination tasks.
          </section>

          <section anchor="UCC_Tenant_ID" title="Tenant ID">
            The Tenant-ID is classifying consumers (tenants) of Cloud
            Services. It is a 6 byte long ID that is defined and managed by
            the Cloud Provider. Similar to the Service-ID the Tenant-ID
            incorporates metadata specific to that tenant. The MetaData field
            is of variable length and can be defined by the Cloud Provider as
            needed.
          </section>
        </section>

        <section anchor="UCC_Extensions_Header" title="UCC Extension Header">
          The UCC proposal <xref target="UCC"/> defines an IPv6 hop-by-hop
          extension header to incorporate the Cloud-ID, Service-ID and
          Tenant-ID. Each ID area also includes bits to define enroute
          behavior for devices understanding/not-understanding the newly
          defined hop-by-hop extension header. This is useful for legacy
          devices on the Internet to avoid drops the packet but forward it
          without processing the added IPv6 extension header. <figure
              anchor="UCCheader" title="UCC IPv6 hop-by-hop extension header">
              <artwork align="center"><![CDATA[

0 8 16 24 32 40 48 56 64
+-------+-------+-------+-------+-------+-------+-------+-------+ | FLAG
| SIZE | CLOUD-ID | FLAG | SIZE |
+-------+-------+-------+-------+-------+-------+-------+-------+ |
SERVICE-ID | FLAG | SIZE |
+-------+-------+-------+-------+-------+-------+-------+-------+ |
TENANT-ID | +-------+-------+-------+-------+-------+-------+
]]\></artwork>
</figure> 
The overall extension header size is 22 bytes.
</section>
      </section>

      <section anchor="B-label-segment-routing"
               title="Policy List in Segment Routing Header">
        The model here suggests using the Policy List described in the
        <xref target="I-D.previdi-6man-segment-routing-header">IPv6 Segment
        Routing Header</xref>. Technically, it would violate that
        specification, as the Policy List is described as containing a set of
        optional addresses representing specific nodes in the SR path, where
        in this case it would be a 128 bit number identifying the tenant or
        other set of communicating nodes.

        <section title="Metaconsiderations">
          <section title="Service offered">
            To Be Supplied
          </section>

          <section title="Pros and Cons">
            <section title="The case in favor of this approach">
              To Be Supplied
            </section>

            <section title="The case against">
              To Be Supplied
            </section>
          </section>

          <section title="Filtering considerations">
            To Be Supplied
          </section>
        </section>
      </section>

      <section anchor="B-label-IID"
               title="RFC 4291 Interface Identifier (IID)">
        The approach starts from the observation that Openstack assigns the
        MAC address used by a VM, and can assign it according to any algorithm
        it chooses. A desire has been expressed to put the tenant identifier
        into the IPv6 address. This would put it into the IID in that address
        without modifying the VM OS or the virtual switch.

        <section anchor="address-format" title="Address Format">
          The proposed address format is identical to the IPv6 <xref
          target="RFC4291">EUI-48 based Address</xref>, and is derived from
          the MAC address space specified in <xref
          target="RFC4862">SLAAC</xref>. However, the MAC address provided by
          the OpenStack Controller differs from an IEEE 802.3 MAC Address.

          Walking through the details, an IEEE 802.3 MAC Address (<xref
          target="MAC"/>) consists of two single bit fields, a 22 bit
          Organizationally Unique Identifier (OUI), and a serial number or
          other NIC-specific identifier. The intention is to create a globally
          unique address, so that the NIC may be used on any LAN in the world
          without colliding with other addresses.

          <figure anchor="MAC"
                  title="Ethernet MAC Address as specified by IEEE 802.3">
            <artwork align="center"><![CDATA[

0 8 16 24 32 40 +-------+-------+-------+-------+-------+-------+
|Organizationally Unique| NIC Specific Number | | Identifier (OUI) | |
+-------+-------+-------+-------+-------+-------+ AA |+---
0=Unicast/1=Multicast +---- 1=Local/0=Global ]]\></artwork>
</figure>

          <xref target="RFC4291"/> describes a transformation from that
          address (which it refers to as an EUI-48 address) to the IID of an
          IPv6 Address (<xref target="addr4291"/>).

          <figure anchor="addr4291" title="RFC 4291 IPv6 Address">
            <artwork align="center"><![CDATA[

0 8 16 24 32 40 48 56
+-------+-------+-------+-------+-------+-------+-------+-------+
|Organizationally Unique| Fixed Value | NIC Specific Number | |
Identifier (OUI) | | |
+-------+-------+-------+-------+-------+-------+-------+-------+ AA
|+--- Reserved +---- 0=Local/1=Global ]]\></artwork>
</figure>

          OpenStack today specifies a local IEEE 802.3 address (bit 6 is
          one). IPv6 addresses are either installed using DHCP or derived from
          the MAC address via SLAAC.

          One could imagine the MAC address used in an OpenStack
          environment including both a tenant identifier and a system number
          on a LAN. If the tenant identifier is 24 bits (it could be longer or
          shorter, but for this document it is treated as 24 bits), as in
          common in VxLAN and QinQ implementations, that would allow for a 22
          bit system number, plus two magic bits specifying a locally defined
          unicast address, as shown in <xref target="TenantMAC"/>.
          Alternatively, the first byte could be some specified values such as
          0xFA (as is common with current OpenStack implementations), followed
          by a 16 bit system number within the subnet.

          <figure anchor="TenantMAC"
                  title="Ethernet MAC Address as installed by OpenStack">
            <artwork align="center"><![CDATA[

0 8 16 24 32 40 47 +-------+-------+-------+-------+-------+-------+
|System Number within | Openstack Tenant | | LAN | Identifier |
+-------+-------+-------+-------+-------+-------+ AA |+---
0=Unicast/1=Multicast +---- 1=Local/0=Global ]]\></artwork>
</figure>

          After being passed through SLAAC, that results in an IID that
          contains the Tenant ID in bits 48..63, has bit 6 zero as a
          locally-specified unicast address, and a 22 bit system number, as in
          <xref target="SLAACIPv6"/>.

          <figure anchor="SLAACIPv6"
                  title="RFC 4291 IPv6 IID Derived from OpenStack MAC Address">
            <artwork align="center"><![CDATA[

0 8 16 24 32 40 48 56 63
+-------+-------+-------+-------+-------+-------+-------+-------+
|system number within | Fixed Value | 24 bit OpenStack | | LAN | |
Tenant Identifier |
+-------+-------+-------+-------+-------+-------+-------+-------+ AA
|+--- Reserved +---- 0=Local/1=Global ]]\></artwork>
</figure>

          More generally, if SLAAC is not in use and addresses are conveyed
          using DHCPv6 or another technology, the IID would be as described in
          <xref target="TenantIPv6"/>.

          <figure anchor="TenantIPv6" title="Generalized Tenant IID">
            <artwork align="center"><![CDATA[

0 8 16 24 32 40 48 56 63
+-------+-------+-------+-------+-------+-------+-------+-------+ |
system number within LAN | 24 bit OpenStack | | | Tenant Identifier |
+-------+-------+-------+-------+-------+-------+-------+-------+ AA
|+--- Reserved +---- 0=Local/1=Global ]]\></artwork>
</figure>

          As noted, the Tenant Identifier might be longer or shorter in a
          given implementation. Specifically in <xref target="TenantIPv6"/>, a
          32 bit Tenant ID would occupy bit positions 32..63, and a 16 bit
          Tenant ID would occupy positions 48..63.

          Following the same model, IPv6 Multicast Addresses can be
          associated with a tenant identifier by placing the tenant identifier
          in the same set of bits and using the remaining bits of the
          Multicast Group ID as the ID within the tenant as show in <xref
          target="TenantMulticast"/>. Flags and scope are as specified in
          <xref target="RFC4291"/> section 2.7. To avoid clashes with
          multicast addresses specified in ibid 2.7.1 and future allocations,
          The Tenant Group ID MUST NOT be zero.

          <figure anchor="TenantMulticast"
                  title="RFC 4291 Multicast Address with Tenant ID">
            <artwork align="center"><![CDATA[

| 8 | 4 | 4 | 88 bits 24 bits | +------
-+----+----+-----------------------------+---------------+
|11111111|flgs|scop| Tenant Group ID | Tenant ID |
+--------+----+----+-----------------------------+---------------+
]]\></artwork>
</figure>
        </section>

        <section title="Metaconsiderations">
          <section title="Service offered">
            The fundamental service offered in this model is that the key
            policy parameter, the Tenant ID, is encoded in every datagram for
            both sender and receiver, and can therefore be tested by sender,
            receiver, or any other party. It is secure in the sense that it
            cannot be directly spoofed; in the sender vSwitch, the vSwitch
            prevents the sender from sending another address as discussed in
            <xref target="bcp38"/>, and if the recipient address is a randomly
            chosen address, even if it meets inter-tenant communication
            policy, there is unlikely to be a matching destination to deliver
            it to. Where this breaks down is if a valid and acceptable
            destination is discovered and used; that is the argument for
            further protection via TLS.
          </section>

          <section title="Pros and Cons">
            <section title="The case in favor of this approach">
              The case in favor of this approach consists of several
              observations: <list style="symbols">
                  Many Cisco customers are using and prefer SLAAC for IPv6
                  clients in OpenStack environments.

                  It is easy to configure this IID from the Neutron
                  Controller without modifying the VM, or it even knowing it
                  happened.

                  From a security perspective, the tenant ID cannot be
                  spoofed per se; the sender of a message is not permitted to
                  send a message from the wrong address or to an unauthorized
                  address.

                  Since it requires no extension headers or other
                  encapsulations, it has no impact on Path MTU.

                  Filters can be applied anywhere, and notably at the
                  sender and the receiver of a message.
                </list>
            </section>

            <section title="The case against">
              In <xref target="I-D.ietf-6man-default-iids"/>, the IETF is
              moving toward deprecating <xref target="RFC4291"/>'s Modified
              EUI-64 IID.
            </section>
          </section>

          <section title="Filtering considerations">
            In this model, the vSwitch needs, for each VM it manages, two
            access control lists: <list style="symbols">
                Zero or more {IPv6 prefix, tenant ID} pairs; these may be
                read as 'if the IPv6 prefix matches {prefix}, the IID contains
                a tenant ID, and it may be {tenant ID}.'

                Zero or more IPv6 prefixes that the VM is authorized to
                communicate without regard to tenant ID.
              </list>

            Generally speaking, one would expect at least one of those
            three lists to contain an entry - at minimum, the VM would be
            authorized to communicate with ::/0, which is to say 'anyone'.

            Among the generic IPv6 prefixes that may be communicated with,
            there may be zero or more <xref target="RFC6052">IPv4-embedded
            IPv6 prefix</xref> prefixes that the VM is permitted to
            communicate with. For example, if the <xref
            target="I-D.ietf-v6ops-siit-dc"/> translation prefix is
            2001:db8:0:1::/96, and the enterprise is using 192.0.2.0/24 as its
            IPv4 address, the filter would contain the prefix
            2001:db8:0:1:0:0:c000:0200/120.

            Note that the lists may also include multicast prefixes as
            specified in <xref target="address-format"/>, such as
            locally-scoped multicast ff01::/104 or locally-scoped multicast
            within the tenant {ff01::/16, tenant id} . While these access
            lists are applied in both directions (as a sender, what prefixes
            may the destination address contain, and as a receiver, what
            prefixes may the source address contain), only the destination
            address may contain multicast addresses. For multicast, therefore,
            the vSwitch should filter <xref target="RFC2710">Multicast
            Listener Discovery</xref> using the multicast subset, permitting
            the VM to only join relevant multicast groups.
          </section>
        </section>
      </section>

      <section anchor="B-5-label-Privacy-Address"
               title="Modified IID using modified Privacy Extension">
        This variant reflects and builds on the <xref target="RFC4291">IPv6
        Addressing Architecture</xref> <xref target="RFC4862">Stateless
        Address Autoconfiguration</xref>, and the associated <xref
        target="RFC4941">Privacy Extensions</xref>. The address format is
        identical to that of <xref target="B-label-IID"/>, with the exception
        that The 'System Number within the LAN' is a random number determined
        by the host rather than being specified by the controller.

        It does have implications, however. It will require <list
            style="symbols">
            a modification to <xref target="RFC4941"/> to have the host
            include the Tenant ID in the IID,

            a means to inform the host of the Tenant ID,

            a means to inform the vSwitch of the Tenant ID,

            a the vSwitch to follow <xref target="RFC6620">FCFS SAVI</xref>
            to learn the addresses being used by the host, and

            a filter to prevent FCFS SAVI from learning addresses that have
            the wrong Tenant ID.
          </list>

        <section title="Metaconsiderations">
          <section title="Service offered">
            To Be Supplied
          </section>

          <section title="Pros and Cons">
            <section title="The case in favor of this approach">
              To Be Supplied
            </section>

            <section title="The case against">
              To Be Supplied
            </section>
          </section>

          <section title="Filtering considerations">
            To Be Supplied
          </section>
        </section>
      </section>
    </section>

</back> </rfc>
