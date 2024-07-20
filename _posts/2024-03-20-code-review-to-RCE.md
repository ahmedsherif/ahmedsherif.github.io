---
title: From Code Analysis to RCE (CVE-2023-22953)
date: 2024-03-20 13:33:37 +/-TTTT
categories: [codereview]
tags: [codereview]     # TAG names should always be lowercase
comments: true
image:
  path: https://ahmedsherif.github.io/assets/img/posts/code-analysis-cover.png
  show_in_post: true
---


## TL;DR: 

In this article, I detail my journey of uncovering CVE-2023-22953 through a meticulous source code review process. I explain the steps involved, including taint analysis, manual code review, runtime debugging, and flow analysis. By following these methods, readers can gain insights into identifying and addressing vulnerabilities within their own codebase effectively.

## Index

- Introduction
- Data flow analysis
  - source & sink concept
- Understanding the code
  - manual code review
  - automated code review
    - SARIF format (SARIF viewer / explorer)
- Spotting the vulnerability
  - Runtime debugging
    - Setting up the environment
    - Verifying the vulnerability
- PHP Insecure deserialization 
  - Exploitation deserialization
  - ROP vs POP 
  - Custom gadget chain
  - **Final Exploit**

