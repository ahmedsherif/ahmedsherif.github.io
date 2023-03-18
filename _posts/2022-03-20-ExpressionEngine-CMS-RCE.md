---
title: ExpressionEngine CMS - From Code Review to RCE
date: 2022-03-20 13:33:37 +/-TTTT
categories: [codereview,CVE-2023-22953]
tags: [codereview, CVE-2023-22953]     # TAG names should always be lowercase
---

## Introduction

In this article, we will dive into the details of a recent vulnerability we discovered in the popular content management system, ExpressionEngine. Specifically, we will focus on a PHP object injection vulnerability, which was identified through a manual source code review. Through this article, I aim to provide a comprehensive understanding of the PHP object injection, including the methods and techniques used to discover it, and the steps taken to find a custom gadget chain, in order to achieve an RCE.

## Manual code review

It is acknowledged that the methodology and approach for conducting manual source code reviews can vary among individuals, and there is currently no standardized method in place. Additionally, the effectiveness of the review may be dependent on the reviewer's familiarity with the programming language, frameworks, and technology used in the application under review.

## sources and sinks

The most common approach while reviewing a code is to look at the paths by starting from the source or the sinks.



> Example of **sources** in PHP:
> - `$_GET[]`
> - `$_POST[]`
> - `$_COOKIE[]`

> Example of interesting **sinks** in PHP:
> - `eval()`
> - `call_user_func()`
> - `require()`
> - `readfile()`
> - `unserialize()`


`system/ee/ExpressionEngine/Service/File/ViewType.php`{: .filepath}

