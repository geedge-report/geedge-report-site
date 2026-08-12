---
title: "Resources"
permalink: /resources/
description: "Resources from the Geedge Networks analysis project: the USENIX Security '26 artifact, forthcoming releases, press reporting, and related academic work."
---

* TOC
{:toc}


## USENIX Security '26 artifact {#artifact}

All of our result files, anonymized PCAPs, and the scripts and tools used to conduct our scans
are archived on Zenodo:

> **<{{ site.paper.artifact }}>**

A top-level README details the artifact and folder structure. Contents include:

| Folder | Contents |
|---|---|
| `tls_length_field_manipulations/scanner` | The length field scanner, with setup instructions |
| `tls_length_field_manipulations/results` | Per-vantage-point results: anonymized PCAPs, aggregated JSON, txt, and csv |
| `quic_fingerprints` | Recorded probes from the QUIC version scan |
| `ip_fragmentation_reassembly` | PCAPs from the IP reassembly scans |
| `ip_fragmentation_reassembly/ipfragtest` | Geneva, used to generate fragmented packets |
| `dns_injection_behavior` | PCAPs for the DNS injector results |
| `dns_injection_behavior/compress-crafter` | Tool used to create the DNS messages |
| `tcp_rst_injection_behavior` | PCAPs from the TCP RST injection fingerprinting scan |
| `tcp_rst_injection_behavior/getkey` | Tool used to derive the seed key |

We also provide the shell scripts used on our vantage points. Test vectors were created by
extending the open-source tool [Censor-Scanner](https://github.com/tls-attacker/Censor-Scanner) as well as using [Geneva](https://github.com/Kkevsterrr/geneva).


## Reporting on the leak {#reporting}

Several news organizations and journalists were granted early access and worked together for
the past year to analyze the files, focusing on non-code documents. Their reporting
covers the export of Geedge Networks' censorship systems to Pakistan, Myanmar, Ethiopia, and
Kazakhstan, focusing on human-rights impacts, procurement networks, and the spread of Great
Firewall–style controls.

- [**The Internet Coup: A technical analysis on how a Chinese company is exporting the Great Firewall to autocratic regimes**](https://interseclab.org/research/the-internet-coup/)  
  InterSecLab · September 2025 · [PDF, 76 pp.](https://interseclab.org/wp-content/uploads/2025/09/The-Internet-Coup_September2025.pdf)

- [**Madlink: A Taiwanese vestige in the Geedge supply chain**](https://interseclab.org/research/madlink-a-taiwanese-vestige-in-the-geedge-supply-chain/)  
  InterSecLab · April 2026

- [**Shadows of control: Censorship and mass surveillance in Pakistan**](https://www.amnesty.org/en/documents/asa33/0206/2025/en/)  
  Amnesty International · September 2025 · [PDF, 102 pp.](https://www.amnesty.org/en/wp-content/uploads/2025/09/ASA3302062025ENGLISH.pdf)

- [**Pakistan: Mass surveillance and censorship machine is fueled by Chinese, European, Emirati and North American companies**](https://www.amnesty.org/en/latest/news/2025/09/pakistan-mass-surveillance-and-censorship-machine-is-fueled-by-chinese-european-emirati-and-north-american-companies/)  
  Amnesty International · September 2025

- [**Silk road of surveillance: The role of China's Geedge Networks and Myanmar telecommunications operators in the junta's digital terror campaign**](https://www.justiceformyanmar.org/stories/silk-road-of-surveillance)  
  Justice for Myanmar · September 2025 · [PDF, 47 pp.](https://jfm-files.s3.us-east-2.amazonaws.com/public/Silk+Road+of+Surveillance+EN.pdf)

- [**Leaked files show a Chinese company is exporting the Great Firewall's censorship technology**](https://www.theglobeandmail.com/world/article-leaked-files-show-a-chinese-company-is-exporting-the-great-firewalls/)  
  The Globe and Mail · September 2025

- [**China exports censorship tech to authoritarian regimes — aided by EU firms**](https://www.ftm.eu/articles/how-china-is-exporting-its-censorship-technology)  
  Follow the Money · September 2025

- [**Wie China seine Great Firewall ins Ausland exportiert**](https://www.derstandard.at/story/3000000286721/wie-china-seine-great-firewall-ins-ausland-exportiert)  
  Der Standard · September 2025
