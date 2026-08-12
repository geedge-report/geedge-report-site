---
title: "Technical Analysis of the Geedge Networks Firewall Source Code Leak"
subtitle: "Our USENIX Security '26 paper: what we found in the leaked source code of Geedge Networks' censorship firewall, and how we tested for it in the wild."
author: "The authors"
description: "Our USENIX Security '26 paper: the first technical analysis of the source code of Tiangou Secure Gateway (TSG), a commercial DPI firewall built by Geedge Networks and linked to the Great Firewall of China."
---

<p class="venue">{{ site.paper.venue }}</p>

<p class="authors">
  {%- for a in site.authors -%}
    {%- if a.site and a.site != "" -%}<a href="{{ a.site }}">{{ a.name }}</a>{%- else -%}{{ a.name }}{%- endif -%}
    <sup>{{ a.aff | join: "," }}</sup>{% unless forloop.last %}, {% endunless %}
  {%- endfor -%}
</p>

<p class="affiliations">
  {%- for pair in site.affiliations -%}
    <sup>{{ pair[0] }}</sup>&nbsp;{{ pair[1] }}{% unless forloop.last %} &nbsp;·&nbsp; {% endunless %}
  {%- endfor -%}
</p>

<div class="badges">
  {% for b in site.paper.badges %}<span class="badge">{{ b }}</span>{% endfor %}
  <span class="badge">Open Science</span>
</div>

<div class="actions">
  <a class="btn btn-primary" href="{{ site.paper.pdf | relative_url }}">Read the paper (PDF)</a>
  <a class="btn" href="{{ site.paper.artifact }}">Artifact &amp; data (Zenodo)</a>
  <a class="btn" href="#walkthrough">Section walkthrough</a>
  <a class="btn" href="#cite">Cite</a>
</div>

<div class="abstract">
  <h2>Abstract</h2>
  <p>
    In September 2025, over 100K internal documents (including code, communications, etc.)
    from Geedge Networks, a Chinese DPI company with ties to the Great Firewall of China,
    were leaked to the public. In this paper, we analyze the source code from this leak,
    focusing on Geedge Networks' flagship product, the Tiangou Secure Gateway (TSG) firewall.
    Working across multiple repositories, we successfully build and run a local copy of
    TSG—revealing key aspects of its architecture, including the protocols it is capable of
    parsing and the format of blocking rules used to censor sites, proxies, and other
    protocols. Finally, we extract several fingerprints from TSG, including custom random
    number generators and parsing idiosyncrasies that allow us to identify its use and
    similar deployments in the Great Firewall of China.
  </p>
  <p>
    This is the <em>first time</em> that the source code of a commercial DPI has been leaked,
    and our work is the first code analysis of a core firewall component used in national
    censorship infrastructure. This unprecedented investigation offers insights that can
    assist circumvention developers and Internet security researchers in further understanding
    the capabilities and limitations of modern censorship technology.
  </p>
</div>

<h2 id="findings">What we found</h2>

<ul class="keys">
  <li>
    <strong>TSG code is running in China's national firewall.</strong>
    <span>We derived fingerprints from the source code first, then measured deployed
    middleboxes. Two independent fingerprints, at different layers of the stack, each land on a
    different GFW component:</span>
    <ul class="evidence">
      <li>
        <strong>DNS.</strong> TSG injects responses with static flags <code>0x8180</code>,
        always encodes the answer name using compression pointers, and follows at most
        <strong>17</strong> pointer jumps when parsing a query. Of the injectors we tested,
        only China's <em>Injector&nbsp;2</em> exhibits all three; Iran and CN Injectors 1 and 3
        match none of them.
      </li>
      <li>
        <strong>TCP.</strong> TSG generates the IPID and window of injected RSTs with a custom
        PRNG keyed by a configurable seed, defaulting to <code>13</code>. Solving for that key
        from observed RSTs returns exactly 13 for both <em>GFW&nbsp;II</em> and
        <em>GFW&nbsp;III</em>. A stable key arising by chance is on the order of
        10<sup>−44</sup>—before accounting for it matching TSG's default value.
      </li>
    </ul>
    <span>Two more results point the same way: <em>GFW&nbsp;II</em> reassembles 102 IP fragments
    against TSG's 101, and its TLS length-field parsing matches an <em>older</em> version of
    TSG's SSL module. Together these indicate the leaked code, or a close derivative, is
    deployed as part of the Great Firewall. The match is not frozen in time: GFW&nbsp;II and
    III now fix TTL at 255 where TSG varies it, and GFW&nbsp;II matched TSG on that field as
    recently as December 2023—so the deployed code has drifted from the leaked version.</span>
  </li>
  <li>
    <strong>The system is built to fingerprint circumvention tools.</strong>
    <span>Signature bundles target Psiphon, Tor, ExpressVPN, NordVPN, Ultrasurf, HideMe,
    Lantern, LetsVPN and many more, using JA3/JA4 hashes, certificate fields, payload lengths,
    byte patterns, and IP ranges—most ranked at the highest risk level.</span>
  </li>
  <li>
    <strong>Blocking is customer-driven and iterative.</strong>
    <span>Jira and Confluence records show deployment sites in Myanmar (440 tickets), Ethiopia
    (53), Kazakhstan and Pakistan reporting failed blocks, setting priorities, and flagging
    collateral damage from over-blocking.</span>
  </li>
  <li>
    <strong>Two of three detection engines are third-party code.</strong>
    <span>Glimpse is built on libprotoident (Libtrace, University of Waikato) with an
    nDPI-derived OpenVPN detector; QDPI wraps Qosmos ixEngine. Comprehensive protocol
    classification is evidently expensive to build even for a well-resourced, state-linked
    vendor.</span>
  </li>
</ul>

<h2 id="walkthrough">The paper, section by section</h2>

<p>
  These pages carry the technical content of the paper for readers who want the findings
  without the PDF.
</p>

<ul class="cards">
  {% assign fs = site.findings | sort: "order" %}
  {% for f in fs %}
  <li class="card">
    <p class="card-meta">§{{ f.paper_section }} · Part {{ forloop.index }} of {{ fs.size }}</p>
    <h3><a href="{{ f.url | relative_url }}">{{ f.title }}</a></h3>
    <p>{{ f.summary }}</p>
  </li>
  {% endfor %}
</ul>

<h2 id="leak">What was in the leak</h2>

<p>
  The company was founded in 2018 by Fang Binxing, colloquially known as the "Father of the
  GFW" for his foundational role in designing the Great Firewall. Geedge also maintains a close
  collaborative relationship with the Massive and Effective Stream Analysis (MESA) Lab at the
  Chinese Academy of Sciences, whose academic research on traffic analysis directly informs
  product development.
</p>

<div class="table-scroll">
<table>
  <thead>
    <tr><th>Category</th><th>Artifact</th><th>Size</th><th>Key contents</th></tr>
  </thead>
  <tbody>
    <tr><td>Source code</td><td><code>mirror/repo.tar</code></td><td>463.0 GiB</td><td>RPM bundles: Firewall, libcbd, QDPI and Glimpse detectors</td></tr>
    <tr><td>Source code</td><td><code>mesalab_git.tar.zst</code></td><td>59.4 GiB</td><td>Git repos: SAPP, Protocol Plugins, Stellar-on-SAPP, Stellar, Maat, tsg-os-buildimage</td></tr>
    <tr><td>Confluence</td><td>2 archives</td><td>46.4 GiB</td><td>VPN signature logs, Psiphon collateral-damage analysis</td></tr>
    <tr><td>Jira</td><td><code>geedge_jira.tar.zst</code></td><td>2.5 GiB</td><td>Deployment tickets (Myanmar, Ethiopia), bug fixes</td></tr>
    <tr><td>Other</td><td>14 files + filelist</td><td>10.7 MiB</td><td>—</td></tr>
  </tbody>
</table>
</div>
<p class="caption">Composition of the 572 GiB Geedge Networks data leak. Note: SAPP and the protocol plugins are additionally packaged in RPM bundles in <code>repo.tar</code>.</p>

<p>
  Civil society investigations have highlighted evidence of deployments of Geedge Networks'
  products across multiple countries, including Kazakhstan (code-named K18/K24), Ethiopia
  (E21), Pakistan (P19), and Myanmar (M22), and describe the company's role as extending
  beyond software provision to encompass system integration, operator training, and ongoing
  technical support. See <a href="{{ '/resources/#reporting' | relative_url }}">Resources</a>
  for that reporting.
</p>

<h2 id="artifact">Artifact and data</h2>

<p>
  Our result files, anonymized PCAPs, scanning scripts, and measurement tools are archived on
  <a href="{{ site.paper.artifact }}">Zenodo</a>. This includes the TLS length-field scanner and
  results, QUIC version probes, IP fragmentation reassembly PCAPs, DNS injection behavior data,
  and the TCP RST injection fingerprinting tool. See
  <a href="{{ '/resources/#artifact' | relative_url }}">Resources</a> for the full contents.
</p>



<h2 id="cite">Cite this work</h2>

<p>
  Please cite the USENIX Security '26 version. A copy of the camera-ready paper is
  <a href="{{ site.paper.pdf | relative_url }}">available here</a>.
</p>

<pre><code>@inproceedings{ablove2026geedge,
  title     = {Technical Analysis of the Geedge Networks Firewall Source Code Leak},
  author    = {Ablove, Anna and Walker, Johnnie and Wolin, Ben and Niere, Niklas and
               Lange, Felix and Ortwein, Aaron and Huremagic, Armin and Priyanka, Richa and
               Zohaib, Ali and Sheffey, Jade and Heitmann, Nico and Halderman, J. Alex and
               Somorovsky, Juraj and Houmansadr, Amir and Ensafi, Roya and Wu, Mingshi and
               Wustrow, Eric},
  booktitle = {35th USENIX Security Symposium (USENIX Security 26)},
  year      = {2026},
  publisher = {USENIX Association}
}</code></pre>
