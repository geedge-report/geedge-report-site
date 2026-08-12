---
title: "Remote Fingerprints & Deployments"
subtitle: "Measuring where TSG code actually runs — DNS, IP, TCP, TLS, and QUIC."
summary: "Five independent fingerprints derived from the source and tested against middleboxes in China, Kazakhstan, Myanmar, Pakistan, and Iran."
paper_section: "6"
order: 4
---

* TOC
{:toc}

Equipped with our functional build of TSG, we explored potential deployments in China,
Kazakhstan, Myanmar, Pakistan, and Iran. For China we separately examined both the GFW and
distinct regional censorship in Henan.

This is not straightforward: analysis of measurements must account for the plug-and-play nature
of TSG components. A given deployment may run the same version of SAPP but a different version
of the SSL plugin. In this context we investigated specific fingerprintable characteristics in
the IP, DNS, TCP, TLS, and QUIC protocols.

**We find high levels of correspondence with GFW DNS *Injector 2* and TCP *GFW II* and *GFW
III*, suggesting that the leaked TSG code (or a close derivative) is indeed deployed as part of
China's national firewall.**

## DNS injector behavior

TSG calls `get_decompressed_name` to extract the queried domain name, supporting both
uncompressed names (length-prefixed label sequences) and compression-pointer–based
representations as defined in RFC 1035. Instead of disallowing forward jumps as RFC 1035
specifies, the function enforces a **fixed limit of 17 pointer jumps per query**; exceeding this
limit causes domain name extraction to fail. TSG can therefore be fingerprinted by crafting two
DNS queries with 17 and 18 pointer jumps: an injection response for the former but not the
latter is indicative of TSG.

We also investigated TSG's DNS response injection code in `build_cheat_pkt`, from which we
extracted two more characteristics: (1) the injected response has static flags of `0x8180`, and
(2) the domain name in the answer section is always encoded using compression pointers.

<div class="table-scroll" markdown="1">

| Device | Flags | Compression | 17-pointer | 18-pointer |
|---|---|---|---|---|
| **TSG** | `0x8180` | ✓ | ✓ | ✗ |
| Iran | `0x81a0` | ✗ | ✗ | ✗ |
| CN Inj. 1 | `0x8580` | ✗ | ✗ | ✗ |
| **CN Inj. 2** | `0x8180` | ✓ | ✓ | ✗ |
| CN Inj. 3 | `0x8400` | ✗ | ✗ | ✗ |

</div>
<p class="caption">Observed DNS injector behavior. Note that in our measurements <em>Injector 3</em> has different flags and no longer uses compression pointers, unlike the initial observation by Anonymous et al.</p>

**Only China's *Injector 2* exhibits the combination of flag value `0x8180`, the use of
compression pointers in injected responses, and the same pointer-handling capability boundary.**
Our results strongly suggest that TSG provides the GFW Injector 2's DNS injection
implementation.

## IP fragmentation reassembly

TSG reassembles IPv4 fragments using `process_ipv4_frag`, which contains an "attack detection"
mechanism that bounds the maximum number of fragments processed per reassembly queue. Due to the
ordering of the check and update of the fragment counter, TSG allows up to **101** IPv4
fragments to be processed for a single queue. The first fragment for which the
`IPV4_FRAG_NUM_PER_IPQ` threshold is exceeded triggers the `IP_FRAG_IGNORE` state, after which
subsequent fragments are ignored.

We used [Geneva](https://geneva.cs.umd.edu/) to send fragmented `ClientHello` messages for
censored domains with varying numbers of IPv4 fragments.

<div class="table-scroll" markdown="1">

| Device | Observed reassembly boundary |
|---|---|
| **TSG** | 101 |
| GFW I | 500* |
| **GFW II** | 102 |
| GFW III | 0 |
| Henan | 0 |
| Kazakhstan | 0 |
| Pakistan (RST) | 0 |
| Pakistan (DROP) | 500* |
| Myanmar | 500* |

</div>
<p class="caption">Observed IP-layer reassembly boundaries. Rows marked 500* indicate reassembly was still successful through 500 fragments.</p>

The observed boundary of **102 fragments for GFW II is very close to TSG's**, suggesting the two
potentially share similar code. Other middleboxes — GFW III, Henan, Kazakhstan, and Pakistan
(RST) — have a boundary of zero, indicating IP fragments are simply forwarded to the next hop.
This is configurable via the `ipv4_reassembly_enabled` parameter in `sapp.toml`.

## TCP RST injector behavior

TSG contains a **custom pseudo-random number generator** for the IPID, TTL, and TCP window size
of injected RST packets, enabled by default via the `signature_mode` parameter.

The function `trick_algo_getrandval_with_seed` generates these fields. The parameters `sip` and
`dip` denote the source and destination addresses of the TCP stream to be terminated; the
parameters `seed_key` and `seed_max_val` are configurable, with defaults of **13** and **65,535**.

When `signature_mode` is enabled, the algorithm couples IPID, `win`, and `val` through
`seed_key`, making it possible to infer `seed_key` from the IPID, TTL, and TCP window fields of
injected RST packets. Because TTL is unstable, we focus only on IPID and TCP window.

Ignoring the corner case where `win == 0`, the algorithm reduces to the following, where
*k* = `seed_key` and *M* = `seed_max_val`:

```
win  = val + (dip mod k)
ipid = M − val · k + (sip mod win)
```

Given two forged packets *i* and *j* with distinct IPID and window values, *k* can be derived by
solving:

```
k = ( (ipid_i − ipid_j) + ((sip mod win_j) − (sip mod win_i)) ) / (win_j − win_i)
```

TSG also hardcodes the setting of the IP Don't Fragment (DF) bit in `deal_ipv4.h`, and can be
configured to send 1–3 RST or RST+ACK packets.

<div class="table-scroll" markdown="1">

| Device | IP DF | TCP flags | Count | Stable *k* |
|---|---|---|---|---|
| **TSG** | ✓ | RST+ACK* | 1–3 | yes (*k* = 13)* |
| GFW I | ✗ | RST | 1 | no |
| **GFW II** | ✓ | RST+ACK | 3 | yes (*k* = 13) |
| **GFW III** | ✓ | RST+ACK | 1 | yes (*k* = 13) |
| Henan | ✗ | RST+ACK | 1 | no |
| Myanmar | ✓ | RST+ACK | 1 | no |
| Pakistan | ✗ | RST+AE+RE | 1 | no |

</div>
<p class="caption">Observed TCP injection fingerprints. "Stable <em>k</em>" indicates whether pairwise differential estimation yields a unique and consistent integer key. *TSG can be configured to send RSTs and not use a seed key.</p>

**GFW II and GFW III consistently yielded *k* = 13, corresponding to TSG's default `seed_key`,
strongly indicating usage of TSG code.** The likelihood of exhibiting a constant seed key by
random chance is approximately 10<sup>−44</sup>, let alone sharing TSG's default value of 13.

Both GFW II and GFW III deviate from TSG in TTL: both fix TTL at 255, while TSG generates TTL
values dependent on an internal state variable `val`, ranging from 48 to 247. Notably, previous
measurements collected in December 2023 *did* show GFW II TTL values consistent with TSG,
suggesting it slightly changed its implementation since then.

In contrast, the other injectors exhibit markedly different behaviors and header values. While
we did not observe the seed key correspondence in other regions, **TSG can be configured to match
the behavior in Myanmar**, which sets the IP DF bit, uses supported RST+ACK flags, and sends 1
packet.

## TLS length field manipulation

We fingerprint how TSG's SSL plugin handles TLS `ClientHello` parsing length fields, which are
prone to ambiguities because some fields are redundant. Where applicable, we set each
`ClientHello` message length field to zero, half its correct value, twice its correct value, and
its maximum value, and tested whether TSG would parse the `ClientHello` and respond by blocking
the connection. Test vectors were created by extending the open-source tool
[Censor-Scanner](https://github.com/tls-attacker/Censor-Scanner).

Each length permutation was sent 20 times from a vantage point in the respective country to a
controlled vantage point in Europe. A permutation was classified as censored only if known
middlebox behavior was observed in at least two-thirds of trials.

<div class="table-scroll" markdown="1">

| Field | TSG | GFW I | GFW II | GFW III | Henan | Myanmar | Kazakhstan | Pakistan (RST) | Pakistan (DROP) |
|---|---|---|---|---|---|---|---|---|---|
| Record Length | ● | ● | ● | ● | ● | ● | ● | ✗ | ✗ |
| Sess. ID / CS Length | ● | ● | ● | ● | ● | ● | ✗ | ✗ | ✗ |
| Comp. Methods Length | ● | ● | ● | ● | ● | ● | ✗ | ✗ | ✗ |
| SNI Name Length | ● | ● | ● | ✗ | ● | ● | ● | ● | ✗ |
| SNI Ext. Length | ● | ● | ● | ● | ▼ | ● | ▼ | ▼ | ✗ |
| Message Length | ▼ | ● | ▼ | ● | ● | ✗ | ✗ | ✗ | ✗ |
| Extensions Length | ✗* / ▼ | ▼ | ▼ | ▼ | ✗ | ▼ | ✗ | ▼ | ✗ |
| SNI List Length | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ | ● | ✗ | ✗ |

</div>
<p class="caption">● bypasses censor · ✗ blocked · ▼ varied. *This blocked outcome corresponds to an older version of TSG, which aligns with the observed parsing behavior of GFW II.</p>

No middlebox exhibited exactly the same behavior as the latest leaked TSG. **GFW II shows
behavior equivalent to an older version of TSG's SSL module**, which suggests usage of that
older version in GFW II's TLS parsing. Kazakhstan and Pakistan (DROP) support potential TSG
deployments, as their generic censorship mechanism of packet drops makes bypass failures
inconclusive.

## QUIC fingerprints

TSG follows the QUIC specification in enforcing the 1200-byte minimum QUIC Initial datagram size
— it only parses QUIC Initial packets of at least 1200 bytes. TSG also supports
pre-standardization QUIC drafts, including Google QUIC versions. It can parse both plaintext and
encrypted QUIC Client Initial packets, and can reassemble fragmented CRYPTO frames, which may
span multiple UDP datagrams.

<div class="table-scroll" markdown="1">

| Field | TSG | GFW | KZ | MM | PK |
|---|---|---|---|---|---|
| **QUIC version** | | | | | |
| GQUIC: Q023–40, 42–45 | ✗ | ✗ | ✗ | ✗ | ✗ |
| GQUIC: Q041, 44–45 | ● | ● | ✗ | ● | ✗ |
| GQUIC: Q046–49 | ● | ● | ✗ | ● | ● |
| GQUIC: T050–51 | ✗ | ✗ | ● | ✗ | ● |
| IETF: draft-22–28 | ✗ | ✗ | ✗ | ✗ | ✗ |
| IETF: draft-29, v1 | ✗ | ✗ | ✗ | ✗ | ✗ |
| IETF: v2 | ● | ● | ● | ● | ✗ |
| **QUIC features** | | | | | |
| Fragmentation | ✗ | ✗ | ✗ | ✗ | ✗ |
| Below 1200 bytes | ● | ✗ | ✗ | ● | ● |
| Forced negotiation | ● | ● | ● | ● | ● |
| Multi-packet Initial | ● | ● | ● | ● | ● |
| **TLS length fields** | | | | | |
| Message Length | ✗ | ✗ | ✗ | ✗ | ✗ |
| Extensions Length | ✗ | ✗ | ✗ | ✗ | ✗ |
| SNI Extension Length | ✗ | ✗ | ✗ | ● | ● |
| SNI List Length | ✗ | ✗ | ✗ | ● | ● |

</div>
<p class="caption">QUIC parser fingerprints, including blocked permutations for TSG's QUIC module. ● bypasses censor · ✗ blocked.</p>

Both Kazakhstan and the GFW block the same test vectors as TSG's QUIC plugin, though we cannot
match them fully due to a difference in supported QUIC versions. **The parsing fingerprint
exhibited in Kazakhstan fully matches an older version of TSG's QUIC parsing code.** The
supported QUIC versions in Myanmar align exactly with those supported by TSG's QUIC, though the
parsing fingerprint mismatch means we cannot confirm its use.

## Measurement ethics

We designed our measurement methodology to have minimal impact on users under censorship in the
analyzed countries. All scans were performed between two vantage points that we own, not
including any third-party server as the target of our scan. During the renting process of all
our vantage points, we made sure that no entity is present on the current EU sanctions list.
