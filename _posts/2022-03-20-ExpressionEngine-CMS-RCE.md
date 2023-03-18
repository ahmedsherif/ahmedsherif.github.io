---
title: ExpressionEngine CMS - From Code Review to RCE
date: 2022-03-20 13:33:37 +/-TTTT
categories: [codereview,CVE-2023-22953]
tags: [codereview, CVE-2023-22953]     # TAG names should always be lowercase
---

## Introduction

In this article, we will dive into the details of a recent vulnerability we discovered in the popular content management system, ExpressionEngine. Specifically, we will focus on a PHP object injection vulnerability, which was identified through a manual source code review. Through this article, I aim to provide a comprehensive understanding of the PHP object injection, including the methods and techniques used to discover it, and the steps taken to find a custom gadget chain, in order to achieve an RCE.

## ExpressionEngine CMS

As described by ExpressionEngine vendor `ExpressionEngine is a flexible, feature-rich, free open-source content management platform that empowers hundreds of thousands of individuals and organizations around the world to easily manage their web site.`

<img width="714" alt="image" src="https://user-images.githubusercontent.com/4347574/226133044-20b16b1a-fb1c-4ca5-8a1b-49a141d1a30a.png">


## Manual code review

It is acknowledged that the methodology and approach for conducting manual source code reviews can vary among individuals, and there is currently no standardized method in place. Additionally, the effectiveness of the review may be dependent on the reviewer's familiarity with the programming language, frameworks, and technology used in the application under review.

## sources and sinks

The most common approach while reviewing a code is to look at the paths by starting from the source or the sinks.

<img width="500" alt="image" src="https://user-images.githubusercontent.com/4347574/226132644-40eb2e6e-fe1a-4978-96d7-867732078851.png">


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

## Looking at the source code 

After cloning the project and looking around for the interesting **sinks**, there was a snippet that caught my eyes to look deeper into which is located at the following path:  `system/ee/ExpressionEngine/Service/File/ViewType.php`{: .filepath}

<img width="500" alt="image" src="https://user-images.githubusercontent.com/4347574/226133562-a65d0bcf-9ea1-4880-ab35-75a61aa44c1d.png">

tracing back the source which is used to be parsed in the `unserialize()` function, it was found to be coming from the cookie input and the cookie name has to be `exp_viewtype`

<img width="500" alt="image" src="https://user-images.githubusercontent.com/4347574/226133669-2f5d0c2b-7c8f-42cf-8731-e751e916c602.png">

## Runtime debugging 

Now it is the time to poke around and test our inputs in the runtime, my prefered way to do so is through remote debugging by dockerizing the applications, setting-up debugging modules and connecting to debugger on VSCode. 

### VSCode configuration

The settings for VSCode and docker to setup remote debugging is quite easy, you can add configuration in VSCode with the following: 

```json

```

<img width="500" alt="image" src="https://user-images.githubusercontent.com/4347574/226133777-cf88cdcc-baee-4b37-9d0e-c317b27c87fe.png">

### Docker configuration 

There are many docker templates which you can use to dockerize php application with database, you can use: https://github.com/amalendukundu/myonlineedu-php-mysql-docker/tree/master. A little bit of modification to add xdebug while running the app: 

```dockerfile
FROM php:7.2-apache

RUN apt-get update && apt-get install -y
RUN pecl install xdebug-3.1.2
RUN docker-php-ext-install mysqli pdo_mysql
RUN export XDEBUG_SESSION=1
**ADD xdebug.ini /usr/local/etc/php/conf.d/xdebug.ini**

RUN mkdir /app \
 && mkdir /app/ExpressionEngine \
 && mkdir /app/ExpressionEngine/www

COPY www/ /app/ExpressionEngine/www/

RUN cp -r /app/ExpressionEngine/www/* /var/www/html/.
```

Adding xdebug.ini as specified above with the following configuration: 

```dosini
zend_extension=xdebug

[xdebug]
xdebug.mode=develop,debug
#xdebug.discover_client_host=1
xdebug.client_port = 9003
xdebug.start_with_request=yes
xdebug.client_host=host.docker.internal
xdebug.log='/var/logs/xdebug/xdebug.log'
xdebug.connect_timeout_ms=2000

```

### Running the application and testing our theories

Now I deployed the application with remote debugging on, it should be connecting back to our host machine on VScode. 
