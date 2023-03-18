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



`system/ee/ExpressionEngine/Service/File/ViewType.php`{: .filepath}

```shell
echo 'No more line numbers!'
x = test;
```


```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
```

```python
import os
import subprocess

# env vars from launch.json 
ghidraHeadless = os.getenv('GHIDRA_HEADLESS')
projectPath = os.getenv('PROJECT_PATH')
projectName = os.getenv('PROJECT_NAME')
binary = os.path.basename(os.getenv('BINARY'))
script =  os.getenv('HEADLESS_SCRIPT')
properties = script.split('.')[0] + '.properties'
properties_template = '''program={program}'''


# Arguments to pass to Ghidra

args = [ghidraHeadless, projectPath, projectName, "-postscript", script]

print(args)

with open(properties, 'w') as f:
    f.write(properties_template.format(program=binary))

with open(properties, 'r') as f:
    print(f.read())

subprocess.run(args)
```