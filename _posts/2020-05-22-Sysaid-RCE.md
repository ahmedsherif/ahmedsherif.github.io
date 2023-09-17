---
title: SysAid On-Premise 20.1.11 - Unauthenticated RCE
date: 2020-05-22 13:33:37 +/-TTTT
categories: [cve-2020-10569]
tags: [ghostcat,cve-2020-10569]     # TAG names should always be lowercase
image:
  path: https://github.com/ahmedsherif/ahmedsherif.github.io/assets/4347574/630813d9-7d2e-4e67-9946-02ddff5bef7e
  show_in_post: true
---

## Introduction

In 2020, I identified a vulnerability in the "SysAid Help Desk Software". This security flaw was significant enough to be assigned the reference CVE-2020-10569. In this article, I'll detail my discovery process, explain the nature of the vulnerability, and discuss its implications for users of the software.

## Sys-Aid Help Desk software

Sys-aid help desk is a software to help corporations with automating and tracking tickets, in addition to reporting and asset management. The product has both SaaS and on-prem solutions, we aim to target the on-prem solutions in this case. 

![image](https://github.com/ahmedsherif/ahmedsherif.github.io/assets/4347574/f676188a-3b1d-4d96-ae0e-f434de96234d)



## Discovery

During a red team engagement, I encountered a specific login page.

![image](https://github.com/ahmedsherif/ahmedsherif.github.io/assets/4347574/87d1e759-a5f6-4525-8f0d-8d471771de91)


At first glance, I was unfamiliar with the product behind this login page. However, a brief online research provided me with a clearer understanding of its functionalities and significance. In red teaming, maintaining a low profile is crucial to avoid detection. Notably, the `.jsp` file extensions indicated that the application was Java-based. Additionally, the presence of the spring icon hinted at the potential use of the `springboot` framework for its development.

## Vulnerability discovery

While there are numerous attack vectors to consider when attempting to compromise an asset, it's essential to opt for methods that minimize the risk of detection. Rather than directly targeting the application, I shifted my focus to the underlying hosting server, which often presents overlooked vulnerabilities.

One such vulnerability that crossed my mind was the GhostCat attack. To validate this hypothesis, I utilized `netcat` to check port `8009`. To my astonishment, the port was indeed open, indicating a potential avenue for further exploration.


## What is GhostCat attack?

> GhostCatcat permits an attacker to access arbitrary files throughout the web application. This includes directories such as `WEB-INF` and `META-INF`, as well as any other location accessible via ServletContext.getResourceAsStream(). Additionally, it enables the attacker to process any file within the web application as if it were JSP.

By default, remote code execution is not achievable. However, if an application on a vulnerable Tomcat version has a file upload flaw, GhostCatcat can be combined with this flaw to facilitate remote code execution. For this to be successful, the attacker needs to save the uploaded files to the document root and have direct access to the AJP port from outside the target network.


## Installing On-prem

![image](https://github.com/ahmedsherif/ahmedsherif.github.io/assets/4347574/c6b54c5a-4618-4ea9-b5e0-19913a192591)
{: width="972" height="589" .w-50 .right}
Luckily, SysAid provide installation of the product with the trial version, therefore, I decided to install it locally and search for files. 

Surfing through the installation directory, we find a very interesting file `UploadIcon.jsp`! The file is used by the administrator to change the icon, Lets try to access the path directly without authentication. 

![image](https://github.com/ahmedsherif/ahmedsherif.github.io/assets/4347574/863f35cf-339f-4d5a-b3fc-2c6822bddb99)
 

Aaaand it works! 

## Recap
So now, we have the following: 

- AJP connector with port `8009` accessible. This helps to read and interpret files from the webroot directory. 
- Unauthenticated file upload on `UploadIcon.jsp` 

## Exploitation

Do you remember how we are able to leverage Local file inclusion (LFI) vulnerabilities in PHP applications to RCE, even with restricted file upload? 

If you can upload anyfile (no matter what extension it is), you still can interpret it with the Local file inclusion vulnerability and get it executed. For example: upload `test.ico` file with PHP code inside and include it through the vulnerability. 

![image](https://github.com/ahmedsherif/ahmedsherif.github.io/assets/4347574/d815de89-aa57-4072-bdc5-5a752fa0055b)

That will be the same case in this application, but we need to know what is the full path of the uploaded file. 

1. Luckily the application shows the full path of the uploaded file in a Javascript function named      `loadEvt`, we can see that the file has the following path: 

![image](https://github.com/ahmedsherif/ahmedsherif.github.io/assets/4347574/0ffdb4d2-c860-4afa-9419-7793bdc1a0d0)


`C:\Program files\SysAidServer\tomcat\webapps\..\..\..\root\WEB-INF\temp\sysaidtemp_[random].dat`

2. Now, the interpretation of the file should be easy, as we will use the AJP connector to interpret the path of `WEB-INF\temp\sysaidtemp_[random].data` instead of `web.xml` file. 

<img width="1247" alt="image" src="https://github.com/ahmedsherif/ahmedsherif.github.io/assets/4347574/734af1e2-c8a7-42d2-a718-7cd5b4188cdb">




3. Enjoy the RCE 
![image](https://github.com/ahmedsherif/ahmedsherif.github.io/assets/4347574/8eaea6b1-3342-4895-a4ec-8f6ee77afe16)

 

### Timeline disclosure

- 5th of April 2020 - Reported both unauthenticated file upload and GhostCatcat attack. 
- 7th of April 2020 - Recieved confirmation from SysAid and fix will be released the next version
- 5th of May 2020 - Sysaid fixed both vulnerabilites and assigned CVE-2020-10569 for it. 
