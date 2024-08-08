---
title: "InmateBridge: Voice Bridge for Prank Calling"
date: 2024-01-06T21:50:16+02:00
description: "InmateBridge is a tool to make prank calls over VoIP"
subtitle: "Prank calling over VoIP with sounds effects"
draft: false
tags: [
    "SIP",
    "telephony",
    "IAX",
    "prank",
    "ruby",
]
categories: [
    "Fun",
]
---

{{< figure src="/images/inmatebridge.png" alt="A painting of an old dirty broken payphone with phone hanging loose in dark prison with a sticker with text 'InmateBridge' on it. There are laughing prisoners surrounding it." >}}

I don't know what it tells about me, but I have a soft spot for prank calls.

My guilty pleasure is listening to the prank call show [Phone Losers of America](https://phonelosers.com/). In the show, the host Brad calls up unsuspecting people with bizarre stories to get a reaction out of them. He's done it for decades, which you can tell from his ability of giving his victims a hard time on the phone while remaining totally cool himself.

Many of his pranks are childish and mean, but some of them I find hilarious.

Over the years, Brad has developed multiple roles he plays using a voice changer and multiple pranks he repeats while slightly twisting them. He also uses many sound effects to boost his pranks.

Many times I have wanted to replicate his pranks and play them on my friends. Other times I have wondered how much more effective social engineering calls would be with a soundboard.

During the Christmas holidays, I wanted to try Ruby and decided to build a system that would let me try out my favorite pranks.

The result is [InmateBridge](https://github.com/ValtteriL/inmatebridge), an application that allows group calling over VoIP and playing sound effects. I included the essential sound effects to enable my favorite pranks and even made a Finnish version of one of them.

## Sound effects

I wanted to include a flushing sound, but unfortunately couldn't find the one Brad uses. Here are the included ones (InmateBridge supports bringing your own):

1. Call from prison
    * {{<audio src="/audio/you-are-receiving-a-call-from-an-inmate.wav">}}{{<audio src="/audio/thank-you-your-call-is-connecting.wav">}}
    * [Original prank](https://youtu.be/C2foeZOuVuU?si=mEqSJOEcBlM6PW0T&t=70)
    * The victim supposedly receives the call from a prisoner. Whatever they choose on the menu will result in call connecting. The call then can continue in any direction. Victims are audibly intimidated and reserved.
2. Call from prison Finnish version
    * {{<audio src="/audio/fi-olet-vastaanottamassa-puhelua-sornaisten-vankilan-selliosastolta.wav">}}{{<audio src="/audio/fi-kiitos-puhelua-yhdistetaan.wav">}}
    * Finnish version of the one above.
3. Buttslam
    * {{<audio src="/audio/buttslam.wav">}}
    * [Long prank, the sound effect comes at the very end](https://www.youtube.com/watch?v=VsfaKNIarp0)
    * When Brad lets victims in on the prank, he may play this sound to make victims think they are on the air on the radio.
4. Funny hold music
    * {{<audio src="/audio/i-wonder-whats-inside-your-bh.wav">}}
    * [Long one](https://www.youtube.com/watch?v=wjcicfMzyuI)
    * Brad is an arrogant customer support rep, who has somehow damaged the victim's service. When the victim wants to talk to the manager, Brad 'transfers' them with this track as the hold music. The ones that remain on the line are furious when they reach the manager Carol (Brad with voice changer).

## Implementation

InmateBridge is deployed as a container, which includes Asterisk PBX and a console UI. The console UI (the Ruby application) is an [Asterisk Stasis application](https://docs.asterisk.org/Asterisk_20_Documentation/API_Documentation/Dialplan_Applications/Stasis/) which communicates with the PBX over REST and Websocket. The PBX needs a SIP trunk to communicate with the telephone network.

Users will call into the bridge using IAX, and use the console UI to play sounds and to initiate calls to victims. Any number of people can call into the bridge and listen to prank calls in action.

{{< figure src="/images/inmatebridge-implementation.png" alt="High-level architecture of InmateBridge" caption="Architecture and workflow of InmateBridge" >}}

## Calling

I got a SIP trunk from Twilio. Then I fired up InmateBridge with the Finnish prison sound effects at my poor little brother, and a couple of friends. I haven't made any cold calling in a while, but I wonder how this would play out as an icebreaker (Probably terribly!).

[![asciicast](https://asciinema.org/a/630473.svg)](https://asciinema.org/a/630473)

Bored and feeling childish? Want to spice up social engineering calls? Want to be a different cold caller or job applicant? Check out [InmateBridge](https://github.com/ValtteriL/inmatebridge)!
