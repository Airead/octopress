---
layout: post
title: "install bugzilla 4.2.4 on ubuntu 12.04 with nginx"
date: 2013-02-01 13:46
comments: true
categories: bugzilla nginx
---

Many people run bugzilla with Apache, but I want to use nginx. So This post mainly shows how to run bugzilla with nginx. To use bugzilla, you should also install mysql to store data, install postfix to send mail and others. Do it with Google.

<!--more-->
## Perpare Web Server
### Install Nginx
to run bugzilla, first a webserver is needed. I chose nginx as my bugzilla's webserver. 

Download ngnix from http://nginx.org/en/download.html, or just get from http://nginx.org/download/nginx-1.2.6.tar.gz. And then unpack it.

<code>
$ wget http://nginx.org/download/nginx-1.2.6.tar.gz  
$ tar xvf nginx-1.2.6.tar.gz  
</code>

cd nginx-1.2.6, run `./configure->make->make install`, and our nginx will be installed at /usr/local/nginx/. If there are some problem, google it. I assume that the nginx has been installed, so next to configure nginx to service for bugzilla.

### Let Nginx Work With Perl Cgi
our nginx should work with perl cgi, beasue the bugzilla is writen by perl. A fastcgi-wrapper.pl is needed. You can get it [**HRER**](/downloads/code/bugzilla/fastcgi-wrapper.pl), or  
`$ wget http://nginxlibrary.com/downloads/perl-fcgi/fastcgi-wrapper`  
(Here I just show the simple method that make ngnix work with perl cgi. See [Setting up Perl FastCGI with Nginx](http://nginxlibrary.com/perl-fastcgi/) For advanced method)

run the perl script with sudo.  
<code>
$ chmod +x fastcgi-wrapper  
$ sudo ./fastcgi-wrapper  
</code>

The error `Can't locate FCGI.pm in @INC ...` maybe occurs.  
Just fix it by `$ sudo apt-get install libcgi-fast-perl`. run fastcgi-wrapper again.

`$ sudo vi /usr/local/nginx/conf/nginx.conf`, clear all contents of nginx.conf and put these as the file contents:
{% include_code lang:ruby bugzilla/nginx_for_bugzilla.conf %}

start nginx  
`$ sudo /usr/local/nginx/sbin/nginx`  
Visit localhost:88 in your browser. If you see Welcome to nginx, that stands for nginx install successfully.

### Test First Perl Cgi
test out a sample Perl script, create a perl script in /usr/local/nginx/html.  
`$ sudo vi /usr/local/nginx/html/index.pl`
{% include_code bugzilla/index.pl %}

Visit localhsot:88/index.pl, the `Perl Environment Variables` will be shown. Ok, the nginx works with perl cgi.

## Install Bugzilla
Download bugzilla-4.2.4.tar.gz at http://ftp.mozilla.org/pub/mozilla.org/webtools/bugzilla-4.2.4.tar.gz. And unpack it.

<code>
$ wget http://ftp.mozilla.org/pub/mozilla.org/webtools/bugzilla-4.2.4.tar.gz  
$ tar xvf bugzilla-4.2.4.tar.gz -C /usr/local/nginx/html
</code>

Now, you can follow the *Bugzilla Quick Start Guide* in bugzilla-4.2.4/README to install.

`/usr/bin/perl install-module.pl --all` will spend long long time. 

If all installations are done, you will visit your bugzilla from loaclhost:88/bugzilla-4.2.4. 

## Referencde
- [Setting up Perl FastCGI with Nginx](http://nginxlibrary.com/perl-fastcgi/)
- [The Bugzilla Guide - 4.2.4 Release](http://www.bugzilla.org/docs/4.2/en/html/)
