---
layout: default
title: About
permalink: /about/
---

# About

Economics student at the University of Tokyo, working on football analytics with tracking and event data.

- GitHub: [napla-dev](https://github.com/napla-dev)
- X (Twitter): [@napladev](https://x.com/napladev)

{% comment %}Put the PDF at assets/pdf/research-summary-ja.pdf; the section appears automatically once the file exists.{% endcomment %}
{% assign summary_pdf = site.static_files | where: "path", "/assets/pdf/research-summary-ja.pdf" | first %}
{% if summary_pdf %}
## Research summary

- [研究要約（日本語・PDF）]({{ summary_pdf.path | relative_url }})
{% endif %}
