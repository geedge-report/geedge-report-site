---
title: "Application Detection & Circumvention Tool Targeting"
subtitle: "How TSG identifies VPNs and circumvention tools — and who asks it to."
summary: "The Context-Based Detector, TSG's VPN signature formats and fingerprints, the third-party engines behind Glimpse and QDPI, and the customer-driven ticket workflow."
paper_section: "4"
order: 2
---

* TOC
{:toc}

TSG has three application detection systems: the Context-Based Detector (CBD), and the Stellar
plugins Glimpse Detector and QDPI Detector. Only the first is Geedge's own work.

## Context-Based Detector (CBD)

CBD, formerly known as AppSketch, is TSG's core application detection system. In earlier
releases AppSketch was implemented as the SAPP plugin `app_sketch_local`, but it has since been
fully integrated into the firewall through the CBD library. Distributed via the `libcbd` RPM,
this library evaluates application fingerprints — known as *signatures*, loaded from Maat.

Built-in signatures are distinguished from user-defined ones by an arbitrary application-ID
boundary (default: 10,000), suggesting that TSG ships with a substantial set of pre-installed
signatures.

### Signature formats

Two primary signature formats appear throughout the leak: **ASW** (likely AppSketch Works) and
**TSG2402**, both defined in the `appsketch-works/asw-controller` repository. The ASW format
appears to be the storage format for signatures: `asw-controller` accepts uploads in TSG2402
format, converts them to ASW, and stores them in a Git repository; exports back to TSG2402 are
also supported. `appsketch-works/app-test-log` is likely one such storage repository, containing
signatures in ASW format.

### Correlation with Maat definitions

CBD loads application signatures from Maat. Although we found no code explicitly converting ASW
or TSG2402 formats into Maat signature definitions, structural similarities between the formats
strongly suggest such a conversion exists. The keys `app_id` and `app_name` in TSG2402 exactly
match Maat expectations, while the TSG2402 `deny_action` corresponds to Maat's
`action_parameter`. We were able to define and block a custom application through specifications
in Maat rules.

## VPN and circumvention tool signatures

One of the most detailed bundles of VPN and circumvention tool fingerprints is in the
Confluence file *VPN 特征文件更新记录* (**VPN Signature File Update Log**), which records a log
of updates to a signature ZIP from March 2024 through November 2024. The signatures are in
TSG2402 format and carry risk levels between three and five.

Critically, the **majority (21 of 29) are ranked five** — the highest risk level. This stands
in sharp contrast to the signatures in `app-test-log`, which target predominantly Chinese
applications and all have a risk score of one.

<div class="table-scroll" markdown="1">

| Risk | Applications |
|---|---|
| **5** | Atlas VPN; Cyber Ghost VPN; Express VPN; Flash VPN; Hide Me VPN; Hotspot Shield VPN; Ivacy VPN; Nord VPN; Opera VPN; Psiphon-Server; TunnelBear VPN; Turbo VPN; Turbo VPN-Payload; Urban VPN; VPN Unlimited; WARP; Windscribe VPN |
| **4** | Gecko VPN; Psiphon-CDN; Psiphon-QUIC; Tor Browser; Ultrasurf VPN |
| **3** | Cyber Ghost VPN-Payload; Refraction Networking; WARP-Enhanced |

</div>
<p class="caption">Risk levels (1–5) for tool signatures included in the VPN Signature File Update Log.</p>

### Select fingerprints

Signatures combine matchable rule attributes of varying complexity: IP addresses, domain names,
JA3/JA4 hashes, SSL certificate fields, payload lengths, and byte patterns. The listed
conditions for each are joined with a logical AND.

<div class="table-scroll" markdown="1">

| Tool | Signature name | Conditions |
|---|---|---|
| Express VPN | `expressvpn_ja3` | SSL JA3 fingerprint match |
| Flash VPN | `flashvpn_cert_issuer` | SSL cert. issuer common name matches `$SUV`; SSL cert. issuer organization matches `$SUV999` |
| HideMe VPN | `hidemevpn_openvpn_udp_payload` | UDP first client→server payload length equals 54 bytes; UDP first client→server payload matches `000000016628` |
| Ultrasurf VPN | `ultrasurfvpn_update_behavior` | Server FQDN matches one of 8 patterns; SSL JA3 fingerprint match |
| Psiphon | `Psiphon-CDN-SSL` | `common.app_id = 68` or `199`; Server FQDN matches `.com`, `.net`, or `.org`; matches 1,023 CDN IP ranges; excludes 56,396 known legitimate FQDNs |

</div>
<p class="caption">A subset of signatures from the VPN Signature File Update Log.</p>

Signatures can also match on the `common.app_id` attribute; another JSON attached to the HTML
file *VPN 特征提取记录* (**VPN Feature Extraction Records**) includes in its application list
many standard network protocols corresponding to this parameter. The Psiphon rule above
specifies values of 68 or 199, which correspond to HTTPS and SSL respectively.

## What drives signature development

To explore the signature development and support workflow, we investigated related Jira
tickets. A case-insensitive search for "vpn" found **700 tickets across 182 issues**. Major
threads correspond with sites in **Myanmar (440 tickets)** and **Ethiopia (53 tickets)**,
extensively detailing continued fingerprinting efforts; frequent topics include reports of
server IP extractions, service prioritization, ineffective signatures and iterative development,
and instances of over-blocking.

**Iterative development and prioritization.** One ticket from the Myanmar site details
ineffective blocking of Signal when the "censorship circumvention" option is enabled, and
highlights previous applicable experience from a "Project K," which likely refers to
Kazakhstan. Blocking of Signal and LetsVPN is noted as higher priority than Lantern, per the
request of the Myanmar client.

**Over-blocking.** A ticket describing an incident of over-blocking in Ethiopia unveils an
interesting design for blocking Psiphon traffic: it suggests that Psiphon3 client IPs
connecting to a popular SNI should be allowed. This is corroborated by the Confluence
documentation file "GTN498 The collateral damage analysis of Psiphon3 Blocking," which
establishes identification of Psiphon3 traffic based on server IP addresses due to the lack of
obvious signatures, and integration of a "whitelist protection mechanism" to avoid
misidentification.

<div class="takeaway" markdown="1">
<span class="label">Takeaway</span>
Circumvention tool signatures are not developed in isolation. We find evidence of iterative,
client-driven processes where customers can report failures, set priorities, and flag collateral
damage.
</div>

## Third-party detection systems

For the remaining two detection systems — the Stellar plugins Glimpse Detector and QDPI
Detector — we find instances of third-party libraries intentionally copied and adapted into the
Geedge Networks code base.

### Glimpse Detector (libprotoident)

The base protocol classification engine in Glimpse is **libprotoident**, a free software project
from the Libtrace team at the University of Waikato, designed to provide a "very limited form of
deep packet inspection," utilizing the leading four bytes of the application payload in both
directions. This is evidenced by the main Stellar session callback subscriber invoking
`lpi_guess_protocol`, which is the libprotoident API for protocol identification. All code
required for this function contains the libprotoident copyright.

**OpenVPN via nDPI.** OpenVPN detection is implemented in `openvpn_identify.cpp`, which appears
to be a modified copy of `openvpn.c` from a 2022 version of nDPI, a GPL-licensed DPI library.
The main function `ndpi_search_openvpn` is renamed to `app_identify_guess_openvpn`, but the
logic remains nearly identical.

**Geedge-added functionality.** Glimpse implements redundant DNS and QUIC protocol
identification in `app_l7_protocol.cpp` and `quic_identify.cpp`, despite these protocols already
being identified and parsed by SAPP protocol plugins. Because Glimpse only publishes an
application ID corresponding to the detected protocol to the firewall, it is unlikely that flows
are parsed further than strictly necessary for protocol identification. Instead, it can be
utilized as part of multi-condition rules that specify protocols.

### QDPI Detector (Qosmos ixEngine)

Unlike Glimpse, QDPI is built off of **Qosmos ixEngine**, a proprietary, enterprise DPI library
which reportedly classifies over 4,700 protocols and applications. This is most clearly
indicated by the Stellar session callback subscriber function eventually calling
`qmdpi_worker_process`, with the inclusion of the associated header `qmdpi.h` appearing below
the comment "Qosmos ixEngine header."

The QDPI RPM contains three shared object files: `qdpi_detector.so`, `libqmengine.so`, and
`libqmbundle.so`. We identified the `libqmbundle.so` as a PB ("protocol bundle") file in a Jira
ticket, which also highlights the strong relationship between QDPI and the AppSketch Database.
We find a comment and segment of code in `qdpi_detector_session.cpp` which exits out of
detection in the case that the AppSketch database is not loaded.

<div class="takeaway" markdown="1">
<span class="label">Takeaway</span>
TSG's reliance on third-party libraries for two of its three detection systems suggests that
comprehensive protocol classification is costly to build from scratch, even for a
well-resourced, state-linked vendor.
</div>

<div class="note" markdown="1">
<span class="label">Disclosure</span>
We disclosed the usage of code by Geedge Networks to both the Libprotoident team and Enea, the
parent company of Qosmos. Both responded to our initial disclosures and we have scheduled
meetings to engage in further discussion.
</div>
