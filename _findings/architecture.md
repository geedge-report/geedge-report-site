---
title: "TSG Architecture"
subtitle: "How the Tiangou Secure Gateway moves a packet from the wire to a blocking decision."
summary: "SAPP packet ingress, protocol plugins, Stellar sessions, the Maat rule engine, and the firewall plugin that enforces policy."
paper_section: "3"
order: 1
---

* TOC
{:toc}

Modern DPI systems generally follow a four-stage workflow: **ingress**, in which the DPI
receives inline or mirrored traffic; **processing**, in which it reassembles flows, parses
protocols, and maintains connection state; **evaluation**, in which it decides whether a
connection matches a censorship policy; and **interference**, in which it prevents, degrades,
or terminates disallowed connections.

TSG spreads these stages across several separately versioned components. A single component
often provides functionality across more than one stage, which is a large part of why the
system is hard to analyze from the outside.

## SAPP: raw packet ingress

Stream Analysis Process Platform (SAPP) is TSG's base packet-processing platform. It was
developed by MESA Labs in 2005 and has gone through four major iterations — `start`, `papp`,
`SAPPv3`, and `SAPPv4` — with the latest available version of TSG built on SAPPv4.

SAPP is a *plugin platform* that separates lower-level packet processing from other system
components. It handles parsing of the L2–L4 layers (Ethernet, IP, UDP/TCP) and of tunneling
and encapsulation protocols (VLAN, PPPoE, VXLAN) before forwarding reassembled packets to
plugins for further parsing and policy evaluation.

### Plugin architecture

SAPP supports three types of plugins, specified and separated by type via the
`conflist.inf` configuration file. Each plugin has its own `*.inf` file registering it to
specific stream types.

| Plugin type | Role |
|---|---|
| Platform | Shared functionality used by other plugins |
| Protocol | Parse application-layer protocols (DNS, SSL, QUIC) and expose extracted fields and events |
| Business | Consume that information and implement specialized application logic |

These plugins are maintained in separate repositories and are distinct from SAPP itself.

### Stream control APIs

SAPP does not initiate blocking, but it provides the underlying functionality to enforce it.
Blocking actions are mediated through SAPP's `stream.h` header, which defines core
stream-manipulation APIs allowing plugins to set stream options, send TCP resets, and inject
arbitrary packets.

The `MESA_set_stream_opt` API gives fine-grained per-stream control at runtime, including:

- `MSO_DROP_STREAM` — discards all subsequent packets associated with a stream
- `MSO_DROP_CURRENT_PKT` — drops only the currently processed packet

Both require TSG to be deployed in-path to actually stop packets.

## SAPP protocol plugins

After receiving SAPP streams, protocol plugins perform protocol detection and targeted
information extraction. Most are defined in the `MESA_Platform` directory of `mesalab_git`.

**DNS plugin.** Registers to UDP SAPP streams. For queries, it parses the domain name, type,
and class. For responses, it extracts all resource and DNSSEC-related records.

**SSL plugin.** Registers to TCP SAPP streams and parses TLS handshakes. It extracts the SNI,
cipher suites, and extension lengths, and computes JA3 and JA4 fingerprint hashes. For
certificates, it parses the subject, issuer, validity period, and Subject Alternate Names.

**QUIC plugin.** Registers to UDP SAPP streams, parses QUIC handshakes, and extracts the SNI,
user agent string, and version information.

<div class="takeaway" markdown="1">
<span class="label">Takeaway</span>
TSG is highly modular. Components are developed and versioned separately — fingerprinting one
plugin does not necessarily imply the presence or version of another in the same deployment.
</div>

## Stellar-on-SAPP: stream ingress

Stellar is a separate system developed by Geedge Networks, written in C++ (SAPP is C), offering
more stateful stream tracking and a richer inter-plugin communication model. Stellar-on-SAPP is
a SAPP platform plugin that bridges the two; its README explicitly identifies it as a transition
solution.

Stellar-on-SAPP consumes stream data produced directly by SAPP — *not* by SAPP protocol plugins
— and converts it into Stellar sessions, which are abstractions of Layer 4–7 connections.

At the time of the leak the `stellar` repository appeared to be under active development but
not yet integrated into TSG: it is not referenced in any build repositories and contains no
tagged releases, though it has commits through late 2024.

Stellar plugins interact through a session-scoped message bus. Plugins create or locate topics,
publish messages to topics, and subscribe to topics via callbacks — allowing multiple plugins to
communicate without the custom glue SAPP would require. Key application detection plugins
include the **QDPI** and **Glimpse** detectors.

## Maat: the rule setting engine

Blocking rules are set in Maat, described in its README as a "unified description framework for
network flow processing configuration." Functionally it is a rule-matching engine: blocking
rules are configured and loaded into *Maat tables*, and plugins invoke Maat to scan extracted
attributes and obtain the corresponding decision or action.

Maat supports three modes for loading rules:

- **Redis** — loads from a Redis server
- **JSON** — loads from a JSON file
- **IRIS** — loads from a combination of a file-based index and data table files

TSG matches on a wide range of scan attributes: transport protocols, application protocols,
tunnel and encapsulation metadata (inner/outer IP, GRE endpoint), and endpoint identity fields
(source and destination IPs, ports, FQDN/SNI, ASN, country). Both UDP and TCP additionally have
attributes for packet payloads, specifically the first client-to-server and server-to-client
packet payloads.

### Example rule

A JSON-mode Maat security rule configured to block the domain `blocked.com`:

<div class="table-scroll" markdown="1">

| Field | Value |
|---|---|
| `rule_table_name` | `SECURITY_RULE` |
| `uuid` | `...00000000A001` |
| `service` | `2` |
| `action` | `Deny` |
| `blacklist_option` | `0` |
| `evaluation_order` | `0.0` |
| **`action_parameter`** | |
| &nbsp;&nbsp;`sub_action` | `drop` |
| &nbsp;&nbsp;`send_tcp_reset` | `1` |
| &nbsp;&nbsp;`after_n_packets` | `0` |
| &nbsp;&nbsp;`send_icmp_unreachable` | `0` |
| **`and_conditions[0]`** | |
| &nbsp;&nbsp;`negate_option` | `false` |
| &nbsp;&nbsp;`attribute_name` | `ATTR_SERVER_FQDN` |
| &nbsp;&nbsp;**`objects[0]`** | |
| &nbsp;&nbsp;&nbsp;&nbsp;`expression` | `blocked.com` |
| &nbsp;&nbsp;&nbsp;&nbsp;`match_method` | `full` |
| &nbsp;&nbsp;&nbsp;&nbsp;`format` | `uncase plain` |

</div>
<p class="caption">Maat security rule for <code>blocked.com</code>. Indentation reflects JSON nesting depth.</p>

## The firewall: administering blocking

The firewall plugin receives parsed traffic attributes from other plugins and passes blocking
decisions. It functions as both a SAPP plugin and a Stellar plugin simultaneously, scanning
packet- and session-level attributes against Maat rule tables and applying the resulting
decisions using the SAPP stream API.

**SAPP plugin interface.** The firewall registers callbacks with each SAPP protocol plugin
(SSL, HTTP, QUIC). When a plugin's callback is invoked, the firewall populates the
corresponding scan attributes with the extracted protocol fields and initiates scanning against
Maat tables.

**Stellar plugin interface.** The firewall tracks session attributes and subscribes to specific
Stellar message topics. At initialization it subscribes to the application-ID message topic
(`TOPIC_APP_ID`); Stellar plugins dedicated to application detection publish to this topic, and
on arrival the firewall updates its scan attributes and scans against Maat tables.

<div class="takeaway" markdown="1">
<span class="label">Takeaway</span>
The firewall aggregates attributes from multiple parallel processing paths (SAPP protocol
plugins, Stellar detectors) before evaluating blocking decisions.
</div>

## Blocking actions

Beyond drops and TCP RSTs, the firewall supports a wide range of actions configurable in Maat
rules.

### Protocol-specific

- **HTTP blocks** — return a specified code (200, 204, 403, or 404) or a redirect to any site with a 302 or 303 code.
- **DNS redirects** — supported.
- **SIP blocking** — inject a "SIP/2.0 480 Temporarily Unavailable" or "SIP/2.0 500 Server Internal Error" message.
- **Mail blocking** — inject either "550 Mail was identified as spam." or "551 User not local; please try &lt;forward-path&gt;."

### Stream-level

- **Throttling** at both session and packet level. The session-level approach initializes a bucket with a set number of bytes per second (`bps`) and drops packets in accordance with its decision; the packet-level limiter drops packets larger than the configured `bps`.
- **Tampering** on every matched packet, or variably depending on sampling parameters. The action swaps the first two and last two bytes of the payload, drops the original packet, and crafts and sends another packet with the modified payload.
- **Deferred blocking** — rules are enforced only after a specified number of packets have been processed.
