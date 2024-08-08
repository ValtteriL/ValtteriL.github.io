---
title: "I made 56874 calls to explore the telephone network. Here's what I found"
date: 2021-06-23T11:06:16+03:00
description: "Post describing my research wardialing Finnish freephones."
tags: [
    "VOIP",
    "SIP",
    "telephony",
    "wardialing"
]
categories: [
    "Research",
]
draft: false
---

>What kind of systems are there in the Finnish telephone network today?

This question appeared to me a while ago when leafing through an old telephone directory from the early 2000s.

There are at least ordinary subscribers, business customer support lines, voice mails, and fax machines.
Subscribers and customer support lines are not of special interest to me.
I am more interested in automated systems that let the caller interact with them somehow.
Voice mails and fax machines fit this definition.
But is there something else as well?

As the telephone network predates the Internet in Finland by over 100 years, the systems in it can be very old and obscure.
Before the Internet gained popularity, these systems were used to provide similar services we use on the internet today.
Would it not be interesting to explore a bit and see what has survived to this day?
As there are no Shodan-like search engines for the telephone network, I needed to do the exploration myself.
This post describes my exploration efforts.

To explore, I used wardialing.
Wardialing is a technique to call a list of telephone numbers automatically using a computer.
Hackers of old used wardialing to explore the telephone network much like port-scanning is used today to explore the Internet.

I used wardialing to call a bunch of numbers and recorded the first 60 seconds of audio received while staying quiet myself.
This produced call recordings I could analyze to learn more about systems in the telephone network.
To my knowledge, nobody has published the results of their wardialing efforts before.
Especially about wardialing Finnish number-spaces or as recent as 2021.

## How I tried to avoid scaring people with ghost calls in the middle of the night

On the receiving end, wardialing causes a ghost call. 
You get a call, which you pick up, but the caller remains silent. 
After a while, the caller hangs up.
This alone can feel threatening to some people.
What if additionally you were woken up by the call in the middle of the night and later you learned that everything you said on phone was recorded, shared with third parties, and published?
I would be angry.
Therefore, I took action to minimize any harm caused to the owners of phone numbers called.

Voice is personal information in itself.
People may also disclose information on the phone they do not want to be shared or published.
To avoid violating people's privacy, I did the following:

- Chose not to publish any recordings where a human answers
- Chose not to disclose any phone numbers
- Ensured that any recording of private individuals did not end up outside the EU, being saved by third parties, or used to train machine learning models
- Deleted recordings and numbers of private individuals after classification

To avoid calling people altogether or flooding emergency services, I chose to call a number space that is typically not used by ordinary subscribers or emergency lines.
I also chose to hide the number I was calling from (displays unknown number at the receiver's end) in hopes that people would not pick up my calls.

I could have further minimized the personal information collection by recording each call for a shorter amount of time.
This however would have made it harder to classify the recordings and recognize the service at the other end.
The 60-second recording was considered optimal as it is enough for the call to be connected and for any recordings to play long enough to make sense of them.

Despite my efforts to not harass private individuals, I woke at least 3 persons in the middle of the night.
This was judged by their announcement of the time when answering or apparent sleepiness.
My apologies to them!

As wardialing was done at a very modest speed, I considered it to not be harmful to the bandwidth of the service providers.

## Accessing the Public Switched Telephone Network (PSTN) on a budget

Since I have no copper Plain Old Telephone Service (POTS) line in my house, I had to look for other ways to wardial.
I discovered a modern wardialer, [WarVOX](https://github.com/rapid7/warvox), that uses VoIP for calling.
I decided to use it.

To work, WarVOX requires a VoIP trunk.
VoIP trunk is a service that acts as a gateway between IP network and the Public Switched Telephone Network (PSTN).
There are a few VoIP trunking services in Finland, but they are either for business customers only, or expensive for only occasional wardialing.
I also assume they would not be happy with wardialing.
Foreign VoIP trunking services are expensive for wardialing Finnish phone numbers.

After learning about illegal termination in the VoIP business, I learned that there is equipment, such as VoIP GSM gateways, that can be used to set up your own VoIP trunk.
Curiously, this equipment comes with the possibility to change the IMSI number.
VoIP GSM gateways act as bridges between IP network and the PSTN network through the GSM network.

With SIM card(s) inside, they connect to the PSTN network like your mobile phone does, allowing you to make and receive calls.
On the IP network, they provide an interface, such as SIP, which can be used to programmatically make and receive calls.

I bought the cheapest VoIP GSM gateway, which provides a SIP interface on the IP network.
WarVOX speaks IAX2, not SIP, so I had to configure Asterisk as a proxy between the two protocols.

To connect to the PSTN network, I bought the cheapest prepaid SIM with a 5 EUR credit.
If the service provider does not like my actions, they can simply block this SIM card.

{{< figure src="/images/topology.png" alt="Wardialing setup topology diagram" caption="VoIP GSM Gateway and Asterisk let WarVOX control calls to the telephone network" >}}

## Choosing numbers to call

Finnish Transport and Communications Agency (Traficom) publishes [decisions regarding telephone network numberings](https://www.traficom.fi/en/communications/broadband-and-telephone/telephone-network-numbering) on their website.
I used the [Finland's E164 numbering plan](https://www.traficom.fi/sites/default/files/media/file/Suomen_numerointisuunnitelma.pdf) as a starting point for choosing target phone numbers to call.
This document from early 2019 specifies the different number spaces and describes their purpose on a high level.

I had the following requirements for the called numbers:

- No premium numbers (my budget was 5 EUR)
- No private individuals (I did not want to annoy and scare people with ghost calls in the middle of the night)
- Maximum amount of systems

In the end, the freephone space (usage "Non-geographic number - Nationwide service numbers (freephone)" on Finland's R164 numbering plan) was determined to be the best candidate.
The subscriber pays a higher fee for the numbers in this space to cover the costs of the callers.
Thus the numbers in this space cost nothing to call so it fits the first requirement.

I imagine the higher fee would keep the number of private individuals at a minimum, as you would not expect private individuals to pay higher fees so that they can be called for free.
Therefore it hypothetically fits the second requirement.

The number of systems in the freephone space is hard to predict.
Having systems in this space would make sense to centralize the costs of using them.

In the numbering plan, there is also a space for Machine-to-Machine communication that sounds like it contains only machines.
Unfortunately, the space is so huge (11 digits) that it is very hard to find numbers that are in use in it.
I initially chose this space but gave up after calling over 10 000 numbers without a single answer.

## Making 56874 calls in 40 days

The bought VoIP GSM Gateway fits only a single SIM card, and it only supports a single line.
Therefore I could only make a single call at a time.
The calls were made randomly to numbers belonging to the freephone space.

WarVOX spent 60 seconds on every call, whether it was answered or not.
This resulted in a wardialing speed of 1 call/minute.
For 56874 calls, this means roughly 40 days calling day and night.

The time required to connect the call and the time it takes for someone to pick up is deducted from the 60 seconds.
This means that the recordings are all shorter than 60 seconds.

I configured the VoIP GSM Gateway to not send Caller ID, so fewer people would pick up and nobody would call me back.

{{< figure src="/images/gsm-voip-gateway.png" alt="GSM VoIP gateway making calls" caption="VoIP GSM Gateway relaying one of the 56874 calls" >}}

## 2% of calls get an interesting answer

After running WarVOX, I was left with audio files containing recordings of 1724 answered calls.
WarVOX identifies Modems and Fax machines automatically and provides signal graphs for all recordings.
The signal graphs can be used to sort the recordings by similarity, but I found this to be poor for classifying and identifying the recordings.

To avoid listening to all recordings myself, used Google Cloud Speech-to-Text to transcribe the recordings, and then used the transcriptions to aside the call recordings that have the same message.
This reduced the number of calls to listen manually to around 300, a bearable number.

I then manually listened to the different recordings and classified them according to the following diagram:

{{< figure src="/images/sorting.png" alt="Call sorting diagram" caption="Calls were sorted according to this diagram" >}}

In the diagram, prospect calls are classified and each classification is marked either not interesting or interesting.
The interesting ones are the answers that let the caller interact with the service somehow, or may let, as in the case of music and unknown systems. 
How many recordings each classification contains is visible in the diagram.

All calls answered by a human were either **businesses** or **private individuals**.
I recognized private individuals from businesses by the informal style of answering the phone.

It is not possible to determine what exactly there are at the other end when there is only **silence** or **ringback** on the wire.
The fact that the recordings contain ringback indicates though that there is a PBX on the receiving end, which picks up and then tries calling an extension.

All recordings that were answered by a machine were recordings of human voice, **fax** machines, music, or unknown (machine).
**Music** contains recordings that contain music for the whole recording.
**Unknown (machine)** contains recordings of calls that contain electromechanical sound.


Recordings of human voice were either **announcements** that announced something (such as number not in use) or asked for input.
Recordings asking for input were either **test numbers**, **voice mails**, **teleconferencing services** or they could not be identified, thus **unknown (waiting for input)**.

In total, only 3% of calls were answered.
Out of these, 69% were identified as interesting.
Thus only 2% of all calls got an interesting answer.

## There are only 74 unique interesting answers

As the large majority of interesting recordings that wait for input were unknown, I decided to classify them further.
The classification was done by identical recording content.
Different recordings inside categories are called variants.
Based on this classification, there are 74 unique interesting answers.

The category of unknown recordings that wait for caller input contained 16 different variants.
There were 2 different test number variants.
The voice mail category contained 23 different variants and there were 24 different variants in the teleconference category.
All fax machines sound the same to my ears, while there were 7 different songs played at my wardialing setup.
Finally, both unknown machines sounded identical to me.

|Category|Number of recordings|Number of variants|
|:-|:-:|-:|
|Unknown (waiting for input)|1093|16|
|Test number|5|2|
|Voice mail|33|23|
|Teleconference|31|24|
|Fax|7|1|
|Music|11|7|
|Unknown (machine)|2|1|

## What might be behind the interesting answers?

I hypothesize that all interesting recordings, except fax machines and unknown machines, are services offered by Private Branch Exchanges (PBXs).
PBXs are telephone systems usually used by businesses that switch calls between multiple users and which can offer additional services such as voice mail and teleconferencing.

There was a large number of answers from **Unknown (waiting for input)**, systems waiting for input that did not hint at what service they provide.
Part of them ask for a PIN code or menu option using a dial-pad, others for voice commands.

The ones asking for PIN could be for example voicemail boxes or teleconferencing services.
Maybe they could be authentications to remote-controlled PBXs too?
The only ways to find out would be to ask someone or trying to brute force the codes.
Brute forcing is however illegal.

The menu option ones could be investigated further by browsing through the options.

The ones asking for voice commands were bots.
There was one Finnish and one English bot.
They could be investigated further by interacting with them.

There were 5 **test numbers**.
These numbers offered 2 distinct kind of test number service:

{{<audio src="/audio/35178.flac">}}
{{<audio src="/audio/14360.flac">}}

The former announces that it is an ICS test platform and proceeds to offer DTMF and FAX testing.
The latter test service seems to belong to Twilio, an American communication platform provider.
It offers DTMF testing, CL readback, echo test, and an SMS menu.

**Voice mail** is still used.
Part of the recordings were business voice mails asking for customer feedback or contact details.
The rest were ordinary ones simply asking for messages after the beep.

**Teleconferencing** was a popular service offered in among the recognized interesting recordings.
Most of them seemed to be offered by businesses.
The exact use cases of the teleconferencing services were not revealed.
They could be company internal or aimed at external users.

As expected, there are still **fax** machines.
Warning, you may not like the sound!
{{<audio src="/audio/5329.flac">}}

11 Calls were answered with only **music**.
The music could indicate something, such as being in queue for some service.
All music variants except for one sounded like generic elevator music that could be the default hold music in PBXs.

Example generic hold music:
{{<audio src="/audio/13645.flac">}}
The exception:
{{<audio src="/audio/16846.flac">}}

Two calls were answered identically by some **unknown machine**.
It was not possible to identify what kind of machines they are from the recording alone.
They sound electromechanical.

## Telco experts: help me identify these answers

There was a single response that was present in 1074 answered calls (91% of all interesting answers) and that waits for the caller to interact with itself.
It says "Tervetuloa palveluun" (Welcome to the service) followed by repeating "Anna tunnusluku" (Please give access code).
The machine does not give any hint of what kind of service it is.

I contacted Finnish telecommunications company Elisa in hopes they could identify this service.
This post will be updated once I get an answer.

**Edit 25.6.2021**: This was identified by a user on [Hacker news](https://news.ycombinator.com/item?id=27602383#27605232) to be the phone management interface of Elisa's [Vakio](https://yrityksille.elisa.fi/ohjeet/vakio-vinkit).
Vakio is a service to distribute business calls to the mobile phones of the employees.
To start getting calls, the employees need to call their private freephone number and give their access code.

Here's one of the recordings:
{{<audio src="/audio/3014.flac">}}

2 unknown machines sound electromechanical.
These mystery machines sound like something out of [Evan Doorbell's](https://www.evan-doorbell.com/) tapes.
I contacted Evan in hopes he will recognize the machines and will update this post in case he does.

{{<audio src="/audio/21873.flac">}}

## Zombie apocalypse, secret messages, misconfigured IVRs, and more

This same recording was played at my wardialing setup on two different numbers.
I was very surprised to hear this bizarre message about the end of the world and a zombie apocalypse when listening to the recordings.

Googling the numbers or the sentences on the recording did not tell me what this is.
The recordings sound like captchas.
Note that I pressed no touch tones!

**Edit 25.6.2021**: A user on [Hacker news](https://news.ycombinator.com/item?id=27602383#27605232) recognized this to be a test number used for integration testing call center integrations for a WebRTC monitoring platform at callstats.io.
As proof, he provided additional information about the recording stating rescue agents taking long tea breaks if one presses 1.
The additional information was correct.
Thus, it seems that the zombie apocalypse was only a joke :-)

{{<audio src="/audio/58512.flac">}}

Maple confirmation message for Clevercrossing. 
There were 6 of these. 
What could this be?
A confirmation message for what?

**Edit 25.6.2021**: I received multiple messages regarding this recording. 
All stated that it is actually saying "Mobile confirmation message for Global Crossing".
Global Crossing was a telecommunications company that went bankrupt back in 2002 and was acquired by Level 3 Communications.
Still no clue what the message is for, though.

{{<audio src="/audio/37348.flac">}}

Funny message speaking English the Swedish way:
{{<audio src="/audio/11852.flac">}}

English speech bot that does not understand silence and is going to escalate my call:
{{<audio src="/audio/30379.flac">}}

Finnish speech bot:
{{<audio src="/audio/67303.flac">}}

Incomplete or misconfigured Finnish IVR menu:
{{<audio src="/audio/30366.flac">}}

SIS - No circuit. There were 2 of these:
{{<audio src="/audio/52178.flac">}}

Nuclear Power Plant status:
{{<audio src="/audio/25898.flac">}}

## What kind of systems are there in the Finnish telephone network?

Based on my results, there are private and business telephones, fax machines, PBXs, and unidentified systems.
The PBXs serve for example as teleconferencing systems, test numbers, and voice mails.
Some PBXs may also have other purposes, but this research was not enough to find out what.

The systems I was not able to identify, such as the unknown machines and the unknown PBX services, could turn out to be something unexpected.
Therefore if you know what they are, please contact me!
You can find a link to the recordings at the end of this post.

I cannot fully answer the original question, because of the narrow number space selection and relatively small sample size.
My exercise thus serves better as a peek into what kind of systems there are in the Finnish freephone number space in 2021.

I only called the numbers and passively listened for 60 seconds.
It would be interesting to attempt interacting with the discovered services somehow and see how they react.

Another idea would be to wardial other phone number spaces with the same methodology.
The E164 numbering plan states that phone numbers of length 5 digits are still in use for older assignments.
Therefore, it is a promising target to discover older services.
As a bonus, this number space is sufficiently small to call through in a reasonable time.
However, this range may contain service numbers that charge premium rates.

## Listen to the raw recordings

All raw recordings marked interesting in the sorting figure, except voice mail, are available [here](https://github.com/ValtteriL/ValtteriL.github.io/tree/master/audio/all).

Voice mails are not published as part of them contain a recording of the owners themselves or the owner's number is being read by the recording.