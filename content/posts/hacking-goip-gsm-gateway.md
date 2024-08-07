---
title: "GoIP-1 GSM gateway could be harnessed for phone fraud by hackers"
date: 2022-02-15T20:25:20+02:00
description: "GoIP-1 GSM gateway contains vulnerabilities that allow hackers to send SMS messages and make calls for free"
tags: [
    "VOIP",
    "SIP",
    "telephony",
]
categories: [
    "Security assessment",
]
draft: false
---

# GoIP-1 GSM gateway could be harnessed for phone fraud by hackers

While listening to [Risky Business ep. 642](https://risky.biz/RB642/), I learned about [a botnet that has been abusing a vulnerability in TP-Link routers to provide SMS messaging as a service for years](https://vblocalhost.com/presentations/from-match-fixing-to-data-exfiltration-a-story-of-messaging-as-a-service-maas/). The exploited vulnerability allowed the botnet operator to send SMS messages on someone else's bill and the operator sold this capability for others, including other criminals. Similar services are no doubt used when you receive smishing messages notifying you about false packages stuck in customs. It is not clear how much this specific botnet operator is making, but there is demand for sure.

The relatively cheap GoIP-1 GSM gateway device I bought specifically for [my wardialing project](/posts/wardialing-finnish-freephones/) has the functionality to send SMS messages and to make calls on the PSTN network. Thus, compromising it would grant a hacker not only the possibility to send smishing messages, but also to make (scam) calls. The device also did not come across as too secure by default so the time investment into hacking one might be small. A device that powerful and easily hackable would be an ideal target for the SMS messaging service operators. 

Intrigued by the business prospect we chose to investigate the device in our “hack weekend” with my good friend and colleague [Lassi Korhonen](https://www.linkedin.com/in/lassi-korhonen-b50510127/). After a weekend of sitting in a dark apartment hacking and occasionally walking to the nearest fast food joint, we uncovered multiple vulnerabilities on the device, some of which enable compromising such devices for the purpose of sending SMS messages or making calls for free. 

{{< figure src="/images/hacking-voip-gsm-gateway-in-the-dark.jpeg" alt="Laptop and GSM gateway in a dark apartment" caption="Sitting in the dark apartment hacking" >}}

## A word about “hack weekends”

Hack weekends are something we started having together with Lassi once we had worked on the security field for some time. The idea is that we choose some target that interests us both and work on it under the same roof over a weekend fueled by pizza and sushi. 

During these weekends we catch up on life and learn from each other while having fun hacking. It doesn't have to be hacking though, we may also focus on developing a proof-of-concept (PoC) of something. Sometimes we find something, sometimes we don't, but we always learn something new. I warmly recommend this format of entertainment for those that are fortunate enough to share a nerdy interest with their friend.

{{< figure src="/images/linnaburger-hacker-lunch.jpg" alt="Linnaburger dish with burger, fries, and coke" caption="On an occasional visit to a nearby fast food joint (Linnaburger in Turku) for lunch" >}}

## Past findings

There have been multiple vulnerabilities discovered in GoIP devices in the past. 

In early 2017, Trustwave [found a secret backdoor telnet account](https://www.trustwave.com/en-us/resources/blogs/spiderlabs-blog/undocumented-backdoor-account-in-dbltek-goip/), 'dbladm', that granted root shell on the device. This account was protected with a custom challenge-response algorithm which in turn relied on obscurity for security. Thus anyone could log in. In our testing, the account seemed to still exist, but the challenge-response algorithm had been changed. Trustwave had reverse-engineered the algorithm from an upgrade package, but based on our research it was no longer part of the newer upgrade packages. Perhaps it could have been extracted from the device with special tooling? We, however, lacked the tools.

In late 2017 the author Securiteam has submitted [multiple vulnerabilities](https://www.exploit-db.com/exploits/44051) reported by an independent security researcher affecting the web panel on GoIP devices to [The Exploit Database](https://www.exploit-db.com/). These vulnerabilities allow downloading the configuration of the device along with all credentials and executing commands on it. Thus they allow full takeover of the device. These vulnerabilities had since been fixed and were of no use against our GoIP-1 device.

Aside from concrete vulnerabilities, GoIP devices are ridden with weak default credentials that users are not forced to change. The weak credentials could most likely be used to gain unauthorized access to the same number of GoIP devices as with the past vulnerabilities (and the new ones disclosed in this blog post).


## Our findings

The firmware version of the GoIP-1 device was GHSFVT-1.1-67-5 M26FBR03A02 RSIM. Before testing, the device was reset to the default configuration, and a static IP was set to the LAN interface. 

The IP address of the device on 'PC' port was 192.168.8.1 and on 'LAN' port 192.168.9.1. The PoCs, therefore, point to these IP addresses. Based on our analysis, all of the vulnerabilities were present in both interfaces of the device.

### Pre-authentication information disclosure

Information that might be useful for an attacker is leaked from the device to unauthenticated users.

#### Ping log

Opening the following URL in browser reveals the ping log for unauthenticated users. This file is only displayed if the ping functionality has been used since rebooting the device.
```
http://192.168.8.1/default/en_US/ping.log
```

#### Call status

Opening the following URL in browser reveals the call status information for unauthenticated users.
```
http://192.168.8.1/default/en_US/dial.xml
```

#### SMS inbox

Opening the following URL in browser reveals all stored SMS messages for unauthenticated users.
```
http://192.168.9.1/default/en_US/include/sms_store.html
```

#### Debug port

The TCP port 2096 on the device is a debug port that allows the eavesdropping of many events. There is no authentication, thus anyone can simply connect to the port and start secretly collecting information. There is also no encryption.

The port leaks for example:

* The phone number associated with the device
* Which phone number the device is calling
* Which phone number calls this device
* The content of SMS messages sent
* The content of all received SMS messages when admin browses to that view in the web panel

The following screenshot shows messages received from the port. Among the messages, it is visible that an SMS message with the content “Hello world!” is sent to a redacted phone number, and that a 5-second test call is made to a redacted phone number.

{{< figure src="/images/goip-debug-port.png" alt="Terminal windows with messages received from goip debug port" caption="SMS messages and phone numbers visible in debug port messages" >}}

### Information disclosure via Local File Inclusion (LFI)

There are two Local File Inclusion (LFI) vulnerabilities in the web panel that allows fetching data useful to an attacker without authentication. Opening the following URLs in browser will display the passwd file of the device.

```
http://192.168.9.1/default/en_US/frame.html?content=..%2f..%2f..%2f ..%2f..%2fetc%2fpasswd
http://192.168.9.1/default/en_US/frame.A100.html?sidebar=..%2f..%2f ..%2f..%2f..%2fetc%2fpasswd
```

These vulnerabilities can be exploited to fetch the serial number, firmware version, and module versions through fetching the file /dev/mtdblock/5.

The most sensitive file an attacker can request is the config.dat, which stores the whole configuration of the device in cleartext. It includes for example all credentials to the device, including SIP, H.323, and web panel. This file can be requested from /tmp/config.dat if the configuration backup feature has been used. Previously the same information was available in /dev/mtdblock/5 according to the [Securiteam's report](https://www.exploit-db.com/exploits/44051). According to our testing, the data was no longer there in this version of the firmware.

Opening the following URLs in browser will fetch the config.dat file from a device that has used the configuration backup feature since the last reboot:
```
http://192.168.9.1/default/en_US/frame.html?content=..%2f..%2f..%2f ..%2f..%2ftmp%2fconfig.dat
http://192.168.9.1/default/en_US/frame.A100.html?sidebar=..%2f..%2f ..%2f..%2f..%2ftmp%2fconfig.dat
```

### Cross-site scripting (XSS)

There were systematic reflected Cross-site scripting (XSS) vulnerabilities in the web panel. Exploiting them is not the most straightforward way for an attacker to gain access to the SMS sending or calling capabilities of the device, therefore I will only report the vulnerable paths and corresponding parameters.

#### Reflected XSS

The following paths contain parameters vulnerable to reflected cross-site scripting. It is possible to provide input that is reflected without sanitization and thus cause Javascript to be executed in the context of the application. The input is however reflected only once and is not saved anywhere. The vulnerable parameters in each path shown in the table below.


|Path|Vulnerable parameter|
|:-|-:|
|/default/en_US/tools.html|type|
|/default/en_US/tools.html|addr|
|/default/en_US/config.html|line1_gsm_imei|
|/default/en_US/config.html|type|
|/default/en_US/ussd_info.html|type|
|/default/en_US/status.html|type|
|/default/en US/sms_info.html|smskey|
|/default/en US/sms_info.html|telnum|
|/default/en US/sms_info.html|smscontent|


#### Stored XSS

The following paths contain parameters vulnerable to stored cross-site scripting. It is possible to provide input that is reflected back without sanitization and thus cause Javascript to be executed in the context of the application. The input is stored by the web panel and reflected to the user every time they visit a certain view until the user deletes or overwrites the malicious input. The vulnerable parameters in each path shown in the table below.

|Path|Vulnerable parameter|
|:-|-:|
|/default/en US/config.html|l1_gsm_oper_type|
|/default/en US/config.html|l1_gsm_bst|
|/default/en US/config.html|sms_mode|
|/default/en US/config.html|rtp_port_lower|
|/default/en US/config.html|lang|
|/default/en US/config.html|time_zone|

### Denial of Service

The same LFI vulnerability used in the Information Disclosure subsection can be used to deny the service of the web panel completely. Requesting the file /etc/ipin will cause the web panel to become so slow that it is no longer usable. Any other user will not be able to use the web panel either. Exploitation does not require authentication. Rebooting the device will fix the condition.

To exploit the vulnerability, open either of the following URLs in your browser:
```
http://192.168.9.1/default/en_US/frame.html?content=..%2f..%2f..%2f ..%2f..%2fetc%2fipin
http://192.168.9.1/default/en_US/frame.A100.html?sidebar=..%2f..%2f ..%2f..%2f..%2fetc%2fipin
```

## How an attacker could use these findings to send free SMS messages or to make free calls?

The attacker profile we consider is someone who can connect to a GoIP-1 device from the internet with only publicly available information and without any higher level of access to the device. Our attacker plans to start SMS messaging or scam calling as a service using GoIP-1 devices, thus they are after the level of access that allows just that.

Based on our analysis, the vulnerabilities disclosed in this blog post create two new avenues for attack for the kind of attacker we consider. The attacker can use the LFI vulnerability to fetch credentials to the device, and then proceed to use the device using those. They could also try to exploit the XSS vulnerabilities to gain the same credentials.

To exploit the LFI vulnerability to get extract credentials from the device, there is a prerequisite that the GoIP configuration has been saved since the last reboot. It is hard to estimate how often the configuration is saved, and thus narrows the window of exploitation. The exploitation of XSS vulnerabilities requires interaction with the victims, which makes exploitation harder.

If these fail, the attacker may resort to credential brute force attacks. There are three well-known users with weak default passwords, the web panel uses basic authentication and is not protected against brute force attacks. This makes it likely a very simple and effective attack vector.

It is unclear whether the findings are valid for also other GoIP devices. The same manufacturer offers GSM gateways with up to 32 channels. In this setting, the number of channels means the number of SIM cards that can be plugged into the device, the number of simultaneous calls that can be made, and the number of different phone numbers one can send SMS messages from. Thus the more channels, the more useful the device is for a hacker. The device used in testing has only one channel and thus is the least useful.

## Recommendations

We strongly recommend against allowing traffic to any port on this device from the public internet or any untrusted device. Given the record of vulnerabilities and the limited time we spent investigating, the device is guaranteed to contain more. Given the clear value of hacking one of these devices, they will likely get targeted.

## Responsible disclosure

These vulnerabilities were reported to the device manufacturer [Dbltek](http://en.dbltek.com/) on 7th November 2021, and the findings were acknowledged. Dbltek was given over 90 days to fix the findings, as well as support for the fixes.

Dbltek has not released a new version of the GoIP-1 firmware after the report. However, looking at the [firmware releases page](http://en.dbltek.com/latestfirmwares.html), we see that the one we tested was not the latest and greatest.  We did not verify whether the vulnerabilities exist in the newest firmware

The device does not have automatic updates, and the update process is not completely straightforward. Thus, it is likely that vulnerable devices are still being sold, and that all devices are not going to be updated. We chose to disclose these details nevertheless to urge the implementation of automatic updates or at least an easier update process. Our findings with the biggest impact also have a limited window of exploitation and thus will not likely put a large number of the devices into immediate danger. By disclosing we also hope to educate potential buyers of the device about the dangers associated with the devices.