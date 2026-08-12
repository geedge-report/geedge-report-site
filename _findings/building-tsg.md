---
title: "Reconstructing TSG in Practice"
subtitle: "Building and running a live copy of the firewall in an isolated lab."
summary: "Locating the build repositories and RPMs, defeating a license check, wiring up a Docker testbed, and the configuration knobs that shape TSG's behavior."
paper_section: "5"
order: 3
---

* TOC
{:toc}

To more deeply characterize the design of TSG, we prioritized developing a functional build we
could run in an isolated lab environment. This section details the technical challenges of that
effort and the key elements of the code base we identified along the way.

## Locating the build

The key repository is `tsg-os-buildimage`, within `mesalab_git`, whose configuration files
(`sapp.toml`, `main.conf`) and RPM versioning live in `manifest.yaml`. Separately,
`mirror/repo.tar` contains RPM bundle versions including varieties with binary files and source
code — and includes several RPMs for repositories that are *not* present in `mesalab_git`.

RPM names include a version number and an abbreviated commit hash, which can be correlated
across the different locations. For example, a `manifest.yaml` entry naming `dns` at version
`2.1.7.1da8dfa` corresponds to git tag `2.1.7` at commit `1da8dfa`, and to the full RPM name
`dns-2.1.7.1da8dfa-1.el8.x86_64.rpm`.

## Building SAPP

To build a functional version of TSG we look to `manifest.yaml` in the latest tagged release of
`tsg-os-buildimage` (`rel-24.10`). The specified SAPP build, `sapp-pr-4.3.67.07feab9`, requires
the issuance of a valid license to run.

Specifically, we identified a `HASP_ENABLED` flag, enabled at build time, which prompts calls to
a wrapper for Sentinel License Development Kit, a commercial software protection solution. This
wrapper spawns a monitoring thread that validates the license once per second. **We circumvented
this check by replacing the original HASP library with a modified wrapper library containing a
stub implementation of the validation function.**

## Configuring SAPP plugins

The main configuration file for SAPP is `sapp.toml`, which specifies a variety of parameters
including deployment modes, packet I/O settings, and secondary configuration files. The latter
includes the SAPP plugin configuration file `conflist.inf`, which we use to load SAPP's DNS,
SSL, and QUIC plugins.

On startup, TSG initializes each protocol plugin in turn — `ssl`, `quic`, `mail`, `http`, `ftp`,
`dns` — reporting per-plugin init timings.

## The testbed

We developed a testbed to evaluate TSG's blocking behaviors in a controlled environment. This
primarily involved configuring SAPP to support the proper traffic flow setup.

SAPP can listen on an interface via the `[packet_io.internal.interface]` table in `sapp.toml`.
It must be run in either:

- **`inline` mode**, which requires that all packets have a VXLAN overlay, or
- **`transparent` mode**, which conducts incomplete L2 forwarding such that MAC addresses are not rewritten.

Given these constraints, we built a Docker setup consisting of a client container, DPI
container, and packet-forwarding container. The client and DPI are both on isolated local
(Docker bridge) networks; the packet-forwarding container rewrites source and destination MAC
addresses so packets can be routed to the Internet. TSG natively supports integration with
Apache Kafka for logging visualizations, so we spun up separate Docker containers hosting the
necessary Kafka endpoints and configured the firewall to visualize a blocked connection.

**We were able to successfully test the DNS, HTTP, TLS, and QUIC protocols and trigger injection
responses, packet drops, and deferred blocking actions.** Through Maat rules, we also defined a
custom app specifying an FQDN matching `blocked.com`, which surfaced in the Kafka UI as an
`app_transition` to `CustomBlockedApp`, `decoded_as: SSL`, `server_fqdn: blocked.com`.

## RST injector configuration

The firewall has its own configuration file, `main.conf`, which controls the RST injector.

**Flags.** TSG can be configured to send one to three RST packets per injection decision via the
`NUM` option, as well as the flags — either RST or RST+ACK — via the `FLAGS` option, where
`FLAGS=4` corresponds to a RST and `FLAGS=20` corresponds to a RST+ACK. The default
configuration sends one RST+ACK.

**IPID, window, and TTL.** TSG has a signature mode, which sets IPID, TCP window, and TTL using
two seed parameters. These are set or disabled either through `sapp.toml`'s `signature_seed1`,
`signature_seed2`, and `signature_enabled`, or through `main.conf`'s `SEED1` and `SEED2`. **The
default configuration is enabled with values of 13 and 65,535** — a default that turns out to be
directly observable in deployed middleboxes, as described in
[Remote Fingerprints]({{ '/findings/fingerprints/' | relative_url }}).

<div class="note" markdown="1">
<span class="label">Not published</span>
<p>We decided against publication of the leaked source code and of our deployment of the firewall
due to ethical concerns. We consider the risk of censors benefiting from such a publication
greater than the potential benefit to the community. We will share our artifact with researchers
or other entities that aim to contribute to the community —
<a href="{{ '/contact/' | relative_url }}">get in touch</a>.</p>
</div>
