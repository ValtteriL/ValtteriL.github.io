---
title: "Ripping Off Professional Criminals by Fermenting Onions - Phishing Darknet Users for Bitcoins"
date: 2023-06-05T18:58:10+03:00
draft: true
description: "Writeup of a tool for creating bitcoin stealing phishing clones of onion services on large scale"
draft: false
tags: [
    "Tor",
    "Phishing",
    "Bitcoin",
    "MitM",
    "Erlang",
]
categories: [
    "Research",
]
---

# Ripping Off Professional Criminals by Fermenting Onions - Phishing Darknet Users for Bitcoins

{{< figure src="/images/a-scary-person-holding-rotten-onion.png" alt="A scary person holding rotten onion" caption="DALL-E's take on \"A scary person holding rotten onion\"" >}}

In 2018, I read about the perfect crime of [stealing the money of credit card fraudsters by making fake carding sites](https://krebsonsecurity.com/2018/05/will-the-real-jokers-stash-come-forward/).

At the time, this felt genius to me; the attackers were apparently making a decent living while nobody was presumably coming after them. (Except maybe now someone will, as they got Krebs'd by Brian). Morally this felt fine as well. It is hard to feel sympathy for fraudsters.

I started to brainstorm similar ways of hacking criminals and making a profit. An idea came to me: making fake darknet sites.

The idea sat unused in my head for a long time while I was busy doing other things. Recently, I got some time on my hands and decided to make a Proof of Concept of it. The result is OF, Onion Fermenter.

What follows is a deep dive that describes the OF implementation and discusses the ethics and legality of operating it.

OF is open source and available for download [here](https://github.com/ValtteriL/OnionFermenter).

## Why phish onion services?

The attackers in the original Krebsonsecurity article were a web design firm. Their methodology used bread and butter web design agency workflow: Designing websites, registering domains, deploying the sites, and perhaps doing some promotion. Then they only needed to wait for unwary fraudsters to stumble across their traps.

While effective, it contains many manual actions. All this also leaves a lot of breadcrumbs that can be followed by investigators like Krebs. Breadcrumbs, such as IP addresses, domain names, and associated information.

Doing the same scam for darknet sites avoids leaving these behind. There are no domains to register, and no IP addresses *should* leak out.

Darknet has further the following advantages for hackers:
- Domains are difficult to remember
- There is no central trusted source for discovering darknet services
- There is no central trusted entity that decides which sites are legitimate
- The services cannot see the source IPs of users
- General slowness and occasional unavailability of services
- Everyone is using virtual currencies when commercing

Inspired by 2FA-resistant reverse proxy phishing kits [Modlishka](https://github.com/drk1wi/Modlishka) and [Evilginx](https://github.com/kgretzky/evilginx2), I planned to exploit these properties as follows: 
1. I publish a reverse proxy as an onion service
2. The reverse proxy relays all traffic between the visitors and the onion service I want to clone
3. The proxy will, on the fly, modify the traffic to not disclose its presence to the visitor or the target darknet service and to redirect any fund transfers to my wallet

The visitor does not remember the onion address of a service. They also cannot easily verify address legitimacy. Thus, they use the one which I planted somewhere they look.

As the cloned service cannot see the origin of their users, they cannot see that many users come from a single source (my onion service). Thus, they may not be alerted to the attack before getting angry feedback from duped users.

This reverse-proxying will cause delay as the connections will pass the Tor network twice. However, as Tor is slow and services are occasionally unavailable, the users will likely not notice it.

And finally, as everyone is using virtual currencies, it is trivial to redirect funds by changing the wallet addresses the cloned service displays to the user. As these transactions are irreversible, once the victim has sent their funds to the wrong address, it is gone for good.

{{< figure src="/images/tor-phishing-using-reverse-proxy-concept.png" alt="Diagram of the concept of phishing darknet users using reverse proxy while stealing bitcoins" caption="Concept diagram of Onion Fermenter or any other reverse proxy Tor phishing tool" >}}

## Darknet phishing in the past

Someone else had the same idea as me. Chris Monteiro has covered two tools for similar darknet phishing: [Onion Cloner from 2014](https://pirate.london/tor-wars-the-onion-cloners-legacy-e28f35a42665) and the more advanced [Rotten Onions from 2016](https://pirate.london/intercepting-drug-deals-charity-and-onionland-a2f9bb306b04).

Rotten Onions was pretty close to my idea. It used the same reverse proxy MitM principle. Perhaps thanks to this effort-reducing tactic, they hosted 763 phishing pages simultaneously. Funnily, the creator was, according to Chris, quite flamboyant and bragged about having "secured significant funding from running Rotten Onions".

Neither of these tools was liked on the /r/onions subreddit. People were trying to [crash the Onion Cloner](https://www.reddit.com/r/onions/comments/2al9i7/crashing_the_onion_cloner/) together by flooding it. The [Rotten Onions author](https://www.reddit.com/r/onions/comments/44lt5g/rotten_onions_a_spiritual_successor_to_onion/) was cursed and later suspended.

## Onion Fermenter (OF)

I created a PoC traffic modifying reverse proxy using Erlang and put it into an isolated network behind a Tor router. I named this setup Onion Fermenter.

The Tor router was configured to host onion services and redirect the traffic to the Erlang application. The application relayed modified traffic to the targeted onion service.

This worked well, but scaling it up was a problem. Adding new services or targets meant a reconfiguration of the Tor router and the Erlang application. Tor daemon also started having issues bootstrapping after serving over 10 services.

Another tedious task was deploying it. Erlang is not the nicest to deploy, especially if you constantly need to reconfigure it dynamically.

### Scaling up

I made deployment and scaling easier by packaging the application into a single container. Now it can be run anywhere a container engine runs, and scaling includes only modifying the number of running containers.

Further wrapping the container with Helm, and deploying on Kubernetes makes it trivial to host any number of phishing sites targeting any darknet services.

The resulting code is open source and available for download [here](https://github.com/ValtteriL/OnionFermenter).

{{< figure src="/images/OnionFermenter-architecture-diagram.png" alt="Architecture diagram of Onion Fermenter" caption="Architecture diagram of Onion Fermenter" >}}

## Demo

Creating 3 phishing clones of a service
[![asciicast](https://asciinema.org/a/DQOE7J2ygPQ9tY7rLQSJIlZPs.png)](https://asciinema.org/a/DQOE7J2ygPQ9tY7rLQSJIlZPs)

## Getting visitors (suckers)

{{< figure src="/images/hacker-poisoning-onions.png" alt="Hacker pouring poison to a well filled with onions" caption="Hacker poisoning onions. Judging by the nose, the artist seems to have been inspired by Queen Grimhilde from Snow White" >}}

Ok, we have a bunch of phishing sites of darknet services. How do we get people to use them?

The previous Tor phishing site links were planted around by poisoning wikis and link directories. This likely works still today.

Many Tor search engines take link submissions, such as the Finnish [Ahmia](https://ahmia.fi). Search engines have no simple way of verifying whether your sites are legitimate or not. Thus, they end up in search results. Imagine the good time a legitimate onion service will have fighting tens of thousands of clones in search results.

If you have money, you can also buy ads for your sites from search engines and link directories.

Finally, there are user groups, such as Dread, where one can post links. However, one must establish themselves on these channels before being taken seriously.
How are darknet services defending themselves?
As this type of attack is nothing new, some darknet sites have taken action to defend themselves and their users.

The money laundering service BitBlender [tried to combat](https://www.reddit.com/r/onions/comments/45k2ux/mitm_phishingonion_cloner/) Rotten Onions in 2016 using a script that redirects users to the correct domain. This measure is trivial for the attacker to sidestep. Some users also have scripts disabled, which means the countermeasure will not save them.

Other services insert the correct onion address to pages as an image, which reverse proxies do not change. It is hard to say how many users pay attention to the addresses in the images, though. 

{{< figure src="/images/kerberos-market-phishing-reminder.jpg" alt="Screenshot of phishing notice in Kerberos darknet marketplace" caption="Likely inefficient phishing protection on Kerberos darknet market" >}}

The marketplace Bohemia has perhaps the most effective countermeasure against OF. They use part of the URI as a captcha and thus force users to check the URI. Attacking this with a reverse proxy setup would require target-specific configuration.

{{< figure src="/images/bohemia-market-phishing-check-ddos-protection.jpg" alt="Screenshot of ddos and phishing protection in Bohemia darknet marketplace" caption="Effective phishing protection on Bohemia darknet market" >}}

Another simple and effective countermeasure against the attacks is using a memorable domain with [Onion-Location](https://community.torproject.org/onion-services/advanced/onion-location/) header to redirect users to the onion site. This presumably also boosts the service visibility. Then the problem is just the breadcrumbs left behind and getting users to enter your site using the domain.

## Who are we ripping off here?

Depending on which darknet sites we target, we will be ripping off different people by running OF.

{{< figure src="/images/intimidating-picture-of-shouting-criminal-with-face-tattoos-who-is-enraged-because-his-bitcoins-got-stolen.png" alt="Realistic and intimidating picture of shouting criminal with face tattoos who is enraged because his bitcoins got stolen" caption="Did YOU steal MY bitcoins???! 🫵😤" >}}

### Darknet markets

The number one target type that comes to mind is the darknet market. With these kinds of sites, you will steal money from vendors and their clients.

Drugs being the number one product for sale, you will steal from professional criminals, drug addicts, and recreational users. Professional criminals are in my opinion fair game if you can take the heat, but stealing from recreational users and especially drug addicts is just low. Drug addicts are already in a weak position. Stealing their fix will likely only cause additional pain.

### Crypto tumblers "mixers"

Crypto tumblers and mixers are often scams, and the exceptions are effectively money laundering services. By targeting them you will be stealing from people who want to launder their cryptocurrency and the launder service runner.

### Ransomware leak sites

Many ransomware gangs have sites on the darknet, where they share the data they have stolen. Some of them also auction the data.

By phishing the auction sites you will be stealing from criminals, and their likely-criminal customers.

### Communities

You can have success phishing communities with entry fees or in-forum commerce. What kind of victims you get depends highly on the type of community you target.

## Effects on the darknet scene

It is beneficial for the Feds to undermine trust between malicious actors. Without trust, the actors cannot cooperate effectively, resulting in less efficient crime.

Darknet is already filled with scams that prey on novices. Onion Fermenter further clouds the darknet waters. Finding legitimate sites will be more difficult. Also, funds can get stolen even more often. Thus, the net effect of OF may be positive for the feds.

For darknet users, malicious or not, this is negative. The user experience suffers, and more vigilance is required to operate. On the other hand, phishing has continued for a long time, and it may be that OF will have no effect at all.

## Is phishing criminals legal?

Bear in mind: I am not a lawyer. 

Intercepting traffic, let alone stealing funds, is not legal. Not even if you did it to criminals exclusively. The victims likely will not report you to the authorities, but if your activities come to light some other way, there will be negative consequences.

Apart from these offenses, you serve the same content as your victim service. Depending on the content, this alone may be highly illegal and punishable. Good luck explaining your good intent to the cops after being caught serving a clone of a sexual abuse site.

## Conclusion

Even if the idea of hacking criminals for profit with Onion Fermenter initially seemed a perfect crime, it is still a crime. Despite possibly being useful in fighting crime, OF muddies the waters in the darknet and undermines trust. Therefore, I strongly discourage you from actually putting OF into use.

I hope OF demonstrates how easy it is to attack darknet users with phishing. Such phishing has a long history, and OF does not bring much new aside from automation and scale to it. But now with the attack demystified and the information in the public domain, anyone can launch such attacks. 

Even though the short-term effect of releasing OF may be negative to Tor users, the long-term effect is likely the opposite. In the short term, there will be an increase in phishing attacks and decreased trust between users. Over time users likely get wiser and/or the Tor project or community introduces technical mitigations to the attack.

Personally, I wish that the security of Tor users improves.