---
title: Chosen Plaintext Attack / The case of Jenkins and US export restrictions (CVE-2024-23897)
date: 2024-07-02 13:33:37 +/-TTTT
categories: [crypto]
tags: [crypto,jenkins,hudson]     # TAG names should always be lowercase
image:
  path: https://s2.ezgif.com/tmp/ezgif-2-f4dfe46c67.gif
  show_in_post: true
---

## TL;DR

As a red teamer, you encountered a Jenkins instance that is vulnerable to CVE-2024-23897, which allowed for limited arbitrary file read. Without credentials and with the /script endpoint inaccessible, you sought to leverage this vulnerability by revealing Hudson to decypt the credentials.

## What is CVE-2024-23897?

>CVE-2024-23897 is a critical vulnerability in Jenkins that enables attackers to read arbitrary files on the Jenkins server, potentially leading to remote code execution. This vulnerability stems from the args4j library, which automatically expands file contents into command arguments when an argument starts with the "@" character. 

Although online resources often suggest this vulnerability can result in remote code execution, they frequently fail to explain the specific technical details and POC required for this escalation.


### RCE Conditions:

1. **Resource Root URL:**
   - The "Resource Root URL" functionality is enabled.
   - Attackers can retrieve binary secrets.
   - The attacker requires an API token for a (non-anonymous) user account. This user account does not need to currently possess Overall/Read permission.

2. **"Remember Me" Cookie:**
   - The "Remember me" feature is enabled (default setting).
   - Attackers possess Overall/Read permission, allowing them to read content in files beyond the initial lines.
   - Attackers can retrieve binary secrets.

3. **Remote Code Execution via CSRF Protection Bypass:**
   - Attackers can retrieve binary secrets.
   - The web session ID is not part of CSRF crumbs.

In our case, none of the above conditions were met. Additionally, the advisory mentions limitations regarding reading binary files, as detailed below:

><span style="color:red">**Limitations for reading binary files**</span>
![Jenkins Advisory Limitations of binary reading](https://ahmedsherif.github.io/assets/img/posts/2/jenkins-advisory-binary.png)

[Reference](https://www.jenkins.io/security/advisory/2024-01-24/#SECURITY-3314)

## Available exploit

There are several available POCs and the best one I tried so far was [this one](https://github.com/xaitax/CVE-2024-23897.git)

```bash
git clone https://github.com/xaitax/CVE-2024-23897.git
```


As unauthenticated users, we are limited to reading only the first three lines of any file through the provided commands.

- `who-am-i`: reads the first line. 
- `enable-job`: reads second line. 
- `help`: reads the 3rd line. 

It can also be executed directly using the `jenkins-cli.jar` file, which can be downloaded from the URL `http://target/jnlpJars/jenkins-cli.jar`. For example:
```bash
java -jar jenkins-cli.jar -s http://sherif.com:9091 who-am-i '@/etc/passwd' 2>&1

ERROR: No argument is allowed: root:x:0:0:root:/root:/bin/bash
java -jar jenkins-cli.jar who-am-i
Reports your credential and permissions.

```

> During the actual red team, the target will most likely need to be accessed through tunneling (Proxychains), in which the `jenkins-cli.jar` tool may not go through, but the Python script can be utilized. In such cases, a tool like Proxifier can be employed on a Windows machine to overcome this challenge.
{: .prompt-warning }

## Reading files

Given the limited lines we can access due to the authentication, we must identify files of interest that may partially assist us.

#### The key files that can be tested include:
- `/proc/self/environ`
- `/proc/self/cmdline`
- `${JENKINS_HOME}/credentials.xml` (Jenkins home can be found in `/proc/self/*`)
- `${JENKINS_HOME}/secrets/master.key`
- `${JENKINS_HOME}/secrets/initialAdminPassword`

Even if an attacker can access all the aforementioned files, including the `master.key`, these assets alone would not be sufficient to decrypt the credentials.

The analysis of how Jenkins encrypts and decrypts credentials, as demonstrated in [this script](https://github.com/tweksteen/jenkins-decrypt/blob/master/decrypt.py), reveals that three components are required:

- `master.key`
- `hudson.util.Secret` (binary file)
- `encrypted credential` (e.g., `AQAAABAAAAAQmEZaw8Fv9tPlXWVQye1TR2KgF3p/wGoYs/TEQCmSxkk=`)

TBD How Jenkins do encryption and Decryption

## Analyzing retrieval of binary data

In the context of binary data retrieval, the absence of `hudson.util.Secret` presents a significant challenge. Attempts to retrieve it using the exploit mentioned above will fail, accompanied by the following error:

```python
http://sherif.com:9091 not reachable: 'utf-8' codec can't decode byte 0xc6 in position 10: invalid continuation byte
```

This issue arises due to the presence of non-printable characters. However, modifying line 80 as follows resolves this:
```python
 result = response.hex()
 ```
> During research, it was observed that ippSec encountered similar issues while attempting to solve the builder machine, as highlighted in their YouTube video https://www.youtube.com/watch?v=jYCOIf9Fcmo&t=1427s. 
{: .prompt-info }


### Wireshark analysis
A Wireshark analysis was conducted to examine the retrieval of binary files. Notably, the binary file begins after the byte sequence `643a20`, with a repetitive sequence of `EFBFBD` starting from the 4th byte.


![wireshark-analysis](https://ahmedsherif.github.io/assets/img/posts/2/wireshark-dump.png)

> This was only the case on the testing environment.
{: .prompt-danger}
## EFBFBD and UTF-8 conversion (�)


EF BF BD (in hex), which is the utf-8 encoding of the Unicode character U+FFFD [Replacement characters](https://www.fileformat.info/info/unicode/char/0fffd/index.htm).When decoding sequences of bytes into Unicode characters, a program may encounter a group bytes that is invalid (i.e. it does not correspond to a Unicode character). In such cases, the program has three choices: it can stop decoding and raise an error, silently skip over the invalid group of bytes, or translate the group of bytes into a special marker. This latter scheme allows most of the text to be read, but still leaves an indication that something went wrong.

coming across the blog of [errno](https://www.errno.fr/bruteforcing_CVE-2024-23897.html), it was an amazing inspiration for me. Errno explained the UTF-8 and `EFBFBD` replacement character to brute force and wrote a nice code in rust to crack it. 

## Crack Me If You Can: Thanks to Uncle Sam's Export Rules!
As previously mentioned, the vulnerability is limited by the ability to read only a few lines of the key, making it challenging to crack. However, Errno made a new discovery: due to US export restrictions, the keys are limited to just `128 bits`, or `16 bytes`. 

![US export keys](https://ahmedsherif.github.io/assets/img/posts/2/export-keys-jenkins.png)

This implies that, to achieve successful decryption, it is sufficient to read only the first `16 bytes` of the Hudson file, even if only the initial few lines are accessible.

## Events often unfold contrary to our desires

Having thoroughly reviewed the Errno code and blog post, and comprehending the concept of replacement characters, I was eager to begin deciphering the Hudson and confidentiality keys. Unfortunately, this anticipation was met with disappointment.

Upon examining the initial 16 bytes in my case, it was evident that the `EFBFBD` replacement character was absent.

![Jenkins-hex-hudson](https://ahmedsherif.github.io/assets/img/posts/2/Jenkins-hex-hudson.png)


> The actual bytes of Hudson file starts after `3a20`. The first 32 bytes are output from Jenkins-cli itself and irrelevant.
{: .prompt-tip }

In order to avoid confusion between the bytes of `Hudson` file itself and `jenkins-cli` error, you can apply the below one-liner:

```bash
cat hudson.bin | tail -c +33 | head -c +16 | xxd
```
![hex-hudson-read-16bytes](https://ahmedsherif.github.io/assets/img/posts/2/Jenkins-reading-Hudson-16bytes.png)

### Byte patterns

Well, since we lost hope in our case for finding the replacement characters `EFBFBD`, we could actually assume that the hudson file did not have non-printable characters and is correct, but that was not the case since we could not decrypt our credentials with it. 

The other assumption is that we have a different encoding and replacement character could be anything else, so maybe we can try with repeated bytes?! 

```bash
dd if=hudson.bin bs=1 skip=32 count=16 2>/dev/null | xxd -p | fold -w2 | sort | uniq -c | sort -nr | head -n 1
```

![Hudson-Repeated-byte](https://ahmedsherif.github.io/assets/img/posts/2/Jenkins-repeated-byte.png)

Of course, I was not sure if this approach will work or not but that was the next logical step for me. 

```bash
cat masterkey.bin | xxd -p | tr -d '\n' | xxd -r -p | sha256sum | cut -c1-32 | sed 's/../0x&,/g'
```


```bash
java -jar ~/Downloads/jenkins-cli.jar -s http://127.0.0.1:9091 who-am-i '@/var/jenkins_home/secrets/hudson.util.Secret' 2>&1  | tail -c +33 | head -c 192 | xxd | tee hudson.bin
```
