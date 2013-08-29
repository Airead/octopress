---
layout: post
title: "Using git through a SOCKS proxy (or SSH tunnel)"
date: 2013-08-27 00:37
comments: true
categories: git ssh
---

Git supports two protocols: HTTP and native git (git://). Unfortunately, git over HTTP is terribly slow. As a result, most public repositories don't prominently display an HTTP URL, because most people would prefer the git:// URL.

In a corporate environment, one does not commonly have the ability to open arbitrary ports on the outside internet. In that case, the git:// protocol is effectively unusable without an unrestricted SOCKS proxy.

To work around the problem, I create my own SOCKS proxy via SSH, then use a wrapper script with git. In essence, this allows me to use git through an SSH tunnel. Any repository, anywhere: one tunnel.
<!--more-->

{% blockquote %}
A note on SOCKS and HTTP proxies

There are two common proxy types in the wild: SOCKS and HTTP. Either type can technically proxy any kind of data, but corporate proxies are usually configured to permit only HTTP and HTTPS, and only on known ports. If you don't know what type of proxy your company uses, it's probably HTTP, and it's probably too restrictive to be useful for the native git protocol.

Most software supports the use of an HTTP proxy, but SOCKS proxy support is a little bit unusual. Mozilla Firefox and Thunderbird are notable for their excellent SOCKS v4 and v5 support, including remote DNS.
{% endblockquote %}

## Procedure

### 1. Install prerequisites

To get everything working, you will need these tools:

- git
- socat
- ssh

### 2. Create your SOCKS4 proxy using SSH

If you don't already have an unrestricted SOCKS proxy, you can create one with the standard openssh client. Much as -L and -R handle port forwarding, the -D option creates a SOCKS version 4 proxy:  

`ssh -nNT -D 8119 remote.host`  

This command starts a SOCKS v4 proxy listening on localhost, port 8119. Requests through the proxy will be proxied via the tunnel and emitted by the remote host.  

If you substitute `autossh` for `ssh`, you can use the exact same arguments to create a tunnel that will automatically restart and reauthenticate itself in the event of a lost connection.

### 3. Put together the wrapper script

Git supports HTTP proxies out of the box, but it has no builtin SOCKS support. We need a wrapper script calling socat to provide the support. I got the original script from a blog post by Emil Sit. He was using an HTTP proxy instead of SOCKS.

{% codeblock lang:objc %}
#!/bin/sh
#
# Use socat to proxy git through a SOCKS proxy.
# Useful if you are trying to clone git:// from inside a company.
#
# See http://tinyurl.com/8xvpny for Emil Sit's original HTTP proxy script.
# See http://tinyurl.com/45atuth for updated SOCKS version.
#

# Configuration.
_proxy=localhost
_proxyport=8119

exec socat STDIO SOCKS4:$_proxy:$1:$2,socksport=$_proxyport
{% endcodeblock %}

I put this script in my home directory at `~/bin/git-proxy-wrapper`.  
Remember to make it executable! (`chmod +x ~/bin/git-proxy-wrapper`)

### 4. Set environment variables

Git checks the GIT_PROXY_COMMAND environment variable when retrieving data. To make git:// work through your SOCKS proxy, set the variable to the git-proxy-wrapper script.

Tcsh:  
`setenv GIT_PROXY_COMMAND ~/bin/git-proxy-wrapper`  

Bash:  
`export GIT_PROXY_COMMAND=~/bin/git-proxy-wrapper`

### 5. Finish up

At this point, git should work with git:// URLs as long as your proxy or ssh proxy session are working. Test it out.

If everything works, you can write the git proxy wrapper setting into your ~/.gitconfig. It's a simple git config command:

git config --global core.gitproxy ~/bin/git-proxy-wrapper

via: [http://www.jones.ec/blogs/a/entry/using_git_through_a_socks](http://www.jones.ec/blogs/a/entry/using_git_through_a_socks)
