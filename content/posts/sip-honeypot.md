---
title: "What I learned of the VOIP hacker scene by setting up a SIP Honeypot"
date: 2021-04-17T18:58:44+03:00
description: "Post describing my study of PBX hackers using SIP honeypot."
tags: [
    "VOIP",
    "honeypot",
    "SIP",
    "telephony",
]
categories: [
    "Research",
]
draft: false
---

I got interested in telephones and the Voice Over IP (VOIP) scene soon after reading Phil Lapsley's Exploding the phone (2013). 
According to the book, there is a whole underground of VOIP hackers.
I had not come across them while lurking in the information security scene.
After my interest sparked, I started paying more attention to telephone-related security research.

The podcast Darknet Diaries by Jack Rhysider has a great first episode called The Phreaky World of PBX Hacking.
This episode describes the modus operandi of Private Branch eXchange (PBX) hackers on a high level while telling the stories of a few convicted PBX hackers. 
According to the episode, PBX hackers make money by first hacking into PBX's and then placing calls to premium numbers they own.

While at the Finnish hacker conference Disobey back in 2019 I saw a presentation Starting from scratch – Mobile Core Network Honeypot by Ms. Sebeni and Dr. Holtmanns.
They wanted to study the methodology of attackers that attack mobile core nodes.
To do this, they set up a node with telco-related services and monitored it for intrusion attempts.

Wanting to learn more about the VOIP hacking scene, I decided to take the same approach as Ms. Sebeni and Dr. Holtmanns.
I wanted to study the methodology of PBX hackers.
Thus, I set up a honeypot PBX, monitored all attacks it faced, and analyzed the data I collected.

## Questions I wanted to be answered

I wanted to answer the following questions:

- Where do the attacks come from?
- How do the attackers try to break into the PBX?
- What do the attackers do after breaking in?

## Setting up PBX with Asterisk and monitoring traffic with Wireshark

To answer the questions, the plan was to set up a PBX, put it on the internet, monitor attacks against it, and finally analyze the collected data.

In VOIP, various protocols could be used to implement calling functionality.
For signaling, there are SIP, H.323, Skinny, IAX, and IAX2.
For media transfer, there are for example RTP, Skinny, and IAX.
Additionally, there are other protocols that are not VOIP specific but that are nevertheless used in VOIP setups.

Many of these protocols could be used to attack a PBX.
Thus, including all of them in the setup would result in a huge attack surface.
This would lead to tedious analysis in the end after having tediously monitored all the protocols.

To keep the attack surface well defined and consistent, I chose to select only a single signaling protocol and a single media transfer protocol.
All other protocols would be left out of the analysis.
For signaling protocol I chose SIP and for media transfer protocol I chose RTP.
Both choices were made because of the ubiquitousness of the protocols.

I decided to use Asterisk as the PBX service, and Wireshark to capture all traffic of the SIP and RTP protocols.
To analyze the traffic captures I used Tshark, Wireshark, and basic Unix utilities.
The PBX was running on a Ubuntu virtual machine in my homelab and it was not advertised in any way.

The honeypot ran in total 5 days, but as the traffic capture it generated was so huge, I decided to analyze only the first 24 hours of it.

### Configuring weak SIP credentials

I set up the following user accounts into Asterisk:

|Extention|Username|Password|
|-|-|-|
|100|||
|110|110|110|
|111|test|test|
|112|112|secret|
|120|alice|password214312aa|

There is a user with extension 100 that requires no authentication at all.
All other users require authentication with a username and password.
The strengths of the credentials vary between different accounts.
All accounts had the same privileges: the privilege to call each other and the privilege to call a number (123) that would play a hello world recording.
None of the accounts had the privilege to call numbers outside the PBX.

## Honeypot under heavy siege from password guessers

In the first 24 hours, Wireshark had captured 381 MB traffic. 
After 5 days, the capture file was over 9GB.
All captured traffic was SIP, there was no RTP traffic.

The following results apply to the first 24 hours of the captured traffic.
A total of 232351 attacks were registered (SIP INVITE or REGISTER requests).
All attacks came from 60 IP addresses.
Most attacks came from United States, Netherlands, and Russia.

There were in total 104564 calling attempts.
Out of these, 95262 calls were dialed to the 5 most called numbers.
103997 calls were placed to the 10 most called numbers.

Only the account with extension 100 was broken into.
45 different IP addresses broke into the account.
1504 calls were attempted to be placed using this hacked account.

The internal hello-world number at extension 123 was not dialed by any attacker.

##### The most guessed usernames

|Username|Number of times guessed|
|:-:|:-:|
|1001|11681|
|1000|9944|
|10|9592|
|11|9582|
|1111|9444|

##### The most called countries

|Prefix code|Country|Number of times called|
|:-:|:-:|:-:|
|+972|Israel|35482|
|+44|United Kingdom|20793|
|+1360|Canada|17813|
|+64|New Zealand|10578|
|+46|Sweden|5236|

##### The most used User-Agents

|User-Agent|Number|
|:-:|:-:|
| |320683|
|friendly-scanner|96241|
|sipcli/v1.8|35520|
|FreePBX 1.8|17813|
|Cisco-SIPGateway|11860|

### Called numbers are test numbers of premium number services

Additional analysis was put to analyze the 10 most called numbers.
All these numbers were found to be test numbers of premium number services.

## Attacker(s) financially motivated, armed with different tools, targeting SIP protocol

Based on the results, SIP services seem to be under heavy attacks.
My PBX was discovered by attackers soon after publishing, and the number of attackers grew constantly.
All attacks used the SIP protocol.
RTP protocol was not used at all in attacks.
The reason RTP traffic was not captured is likely because, in VOIP, a call must first be initiated with signaling protocol (in this case SIP) before RTP traffic begins flowing, and there were no successful calls.

The attack methods of choice were calling through the PBX without authentication and credential guessing.
SIP requests INVITE and REGISTER were used for these.
A misconfigured PBX might let attackers call through it without authentication, whereas PBX with weak user account credentials can fall victim to credential guessing attacks.

The most used usernames all began with digit 1.
I wonder if the attackers try the most used usernames first or whether they simply start from 1 and go up to infinity.
The latter is plausible, as the honeypot was active only for 24 hours, the attackers would still be in process of guessing usernames from the lower end.

The 60 attacking hosts used automated tools to conduct their attacks.
It is hard to say whether the attacker IPs belong to the real attacker(s) or unlucky entities that got hacked.
It is also hard to say how many attackers are really behind the attacks.

Looking at the User-Agents of the SIP clients used, most attacks came from clients that omitted that field completely.
The second most used User-Agent was friendly-scanner which belongs to a popular open-source SIP pentesting tool SIPVicious.
The rest of the User-Agents indicate rather clearly which software they belong to.
It is possible to spoof this field, thus, it cannot be fully trusted.

The 10 most called numbers reveal that the attackers are financially motivated.
The numbers are test numbers that, when receiving calls, let the attackers know a call was successful.
At this point, the attacker would acquire a premium number in the region of the test number and start pumping calls to it through the PBX.
The victim would be responsible for paying the potentially enormous bill.
The fact that most of the numbers called by different attacking hosts were the same, can indicate that the attackers were the same, or that they simply used the same premium number service(s).

Only the account with extension 100 that required no authentication credentials was broken into.
45 different attacker hosts broke into the account.
After breaking in they attempted to make 1504 calls as the user.
This number is quite small compared to the number of calls that were attempted otherwise.
Maybe the attackers simply try a list of numbers to call before giving up and continuing to guess other credentials?

Nobody called the internal number hello-world number I set up.
This was a disappointment, as I was eager to hear what the attackers sound like.

### Answers to my initial questions

SIP services are heavily attacked by credential-guessing financially motivated attackers.
These attackers scan the internet for fresh SIP services.
Their attacks cause a significant amount of traffic to flow to victim SIP services.

To answer the research questions based on the collected data:

>Where do the attacks come from?

Most attacks come from hosts located in United States, Netherlands, and Russia.
The nature of these hosts was not revealed by analyzing their attacks.

>How do the attackers try to break into the PBX?

Attackers try to break into the PBX using mainly brute force. 
Most of the traffic captured were SIP INVITE or REGISTER requests that can be used to guess credentials.
There was a single packet received on the SIP port that did not conform to the protocol.
Analyzing the packet, it did not look like an exploit.

>What do the attackers do after breaking in?

After breaking in, and even before, attackers try to make calls.
Most of the calls were destined for Israel, United Kingdom, Canada, New Zealand, and Sweden.
All the numbers were premium number test numbers.

### Why my results may not tell the whole story?

Because so much traffic was received, I only analyzed data from the first 24 hours.
This may result in distorted results as it takes time for attackers to discover the service.
At the very least it limited the number of attacks and attacking hosts.

As attacks were analyzed only by analyzing the traffic, some attacks may have gone unnoticed.
There could be for instance SIP or RTP packets that try to exploit vulnerabilities in Asterisk or some other PBX implementations.
These attacks went undetected if those exploiting packets conformed to SIP protocol by Wireshark.

### Questions to be answered by future work

It would be interesting to get answers to the following questions:

- Which passwords are used in brute force attacks?
- Are other VOIP protocols attacked? How?
- Are the non-VOIP-specific services attacked by PBX hackers? How?

All these questions could be answered by conducting a similar honeypot study with different configurations.

## How to protect your VOIP setup

To protect your SIP & RTP-based VOIP setup from the attacks my honeypot received, you probably get the best effect using fail2ban or some other brute force stopper.
Fail2ban will ban the attacker IPs after a few failed attempts for a while.
This will stop the simple attack scripts, and at least greatly slow down scripts that are smart enough to adapt to banning.

For more security, simply whitelist the hosts that your VOIP setup is required to communicate with, and drop SIP & RTP packets from all other hosts.

Remember that both of these are considered supplementary security measures.
They will not save you if your PBX is misconfigured to allow arbitrary unauthenticated calls, or if your PBX has weak user account credentials.
