+++
date = '2025-12-23T15:59:59+01:00'
draft = false
hidden = false
title = 'Its All About Defang'
weight = 3
+++

![Ed Skoudis](/images/act1/edskoudis-avatar.gif)

## Objective

| Difficulty | Description |
| ---------- | ----------- |
| 1/5 | Find Ed Skoudis upstairs in City Hall and help him troubleshoot a clever phishing tool in his cozy office. |

## Ed Skoudis mission statement

> I'm the Founder of Counter Hack Innovations, the team that brings you the SANS Holiday Hack Challenge for the last 22 years.
> 
> I'm also the President of the SANS Technology Institute College, which has over 2,300 students studying for their Bachelor's Degrees, Master's Degrees, and various certificates.
> 
> I was the original author of the SANS SEC504 (Incident Handling and Hacker Attacks) and SANS SEC560 (Enterprise Penetration Testing) courses.
> 
> I love Capture the Flag games and puzzles.
> 
> I've got a steampunk office filled with secret rooms, and I collect antique crypto systems and communication technologies.
> 
> I've got an original Enigma machine (A726, from the early war years), a leaf of the Gutenberg Bible (1 John 2:3 to 4:16) from 1455, and a Kryha Liliput from the 1920s.
> 
> ----
>
> Oh gosh, I could talk for hours about this stuff but I really need your help!
> 
> The team has been working on this new SOC tool that helps triage phishing emails...and there are some...issues.
> 
> We have had some pretty sketchy emails coming through and we need to make sure we block ALL of the indicators of compromise. Can you help me? No pressure...

## Its all about the defang terminal

This objective introduces the concept of defanging Indicators of Compromise (IOCs) rather than simply identifying them. The focus is not on detection or attribution, but on understanding how IOCs should be rendered inert when shared publicly, reported externally, or documented in a defensive context.

The exercise reinforces a subtle but important operational habit: IOCs must be transformed in a way that preserves analytical value while preventing accidental execution, resolution, or user interaction. URLs, domains, IP addresses, and email addresses are intentionally modified to break functionality without obscuring meaning. This mirrors standard practice in incident reports, threat intelligence sharing, and training material.

Rather than testing technical skill, the challenge validates awareness of defensive hygiene and safe handling procedures. The provided interface walks through common IOC types and confirms that each has been correctly defanged using accepted conventions.

By completing this step, the player demonstrates familiarity with how malicious artifacts are safely communicated across teams and organizations, setting the tone for later challenges that rely on accurate but non-operational data handling.

### Extract IOCs (Step 1)

#### Domain

```bash
\b(?!(?:\d+\.)+\d+\b)(?:[a-z0-9-]+\.)+(?!(?:exe|corp)\b)[a-z]{2,63}\b
```

* icicleinnovations.mail
* mail.icicleinnovations.mail
* core.icicleinnovations.mail

![Regex domains](/images/act1/act1-defang-1.png)

#### IP addresses

```bash
\b(?!(10)\.)(?:(?:25[0-5]|2[0-4][0-9]|1?\d{1,2})\.){3}(?:25[0-5]|2[0-4][0-9]|1?\d{1,2})\b
```

* 172.16.254.1
* 192.168.1.1

![Regex IPs](/images/act1/act1-defang-2.png)

#### URLs

For this one I just nicked the provided regex for URLs and replaced the start `http` with a fuzzier version `https?`

```bash
https?://[a-zA-Z0-9-]+(\.[a-zA-Z0-9-]+)+(:[0-9]+)?(/[^\s]*)?
```

* https://icicleinnovations.mail/renovation-planner.exe
* https://icicleinnovations.mail/upload_photos

![Regex URLs](/images/act1/act1-defang-3.png)

#### Emails

The provided regex example worked like a charm:

```bash
\b[a-zA-Z0-9._%+-]+@(?:[A-Za-z0-9-]+\.)+(?!corp\b)[A-Za-z]{2,63}\b
```

* sales@icicleinnovations.mail
* info@icicleinnovations.mail

![Regex emails](/images/act1/act1-defang-4.png)

### Defang & Report (Step 2)

```bash
s/\./[.]/g; s/@/[@]/g; s/:\//[://]/g; s/http/hxxp/g
```

![Defang and send report](/images/act1/act1-defang-5.png)

It is worth mentioning that the system helps you in great ways on what should be submitted. For instance, upon submission you are presented with a warning like this if anything is wrong: 

![Defang and send report](/images/act1/act1-defang-7.png)

Upon successful completion we see the report: 

![Defang and send report](/images/act1/act1-defang-6.png)

## Ed Skoudis closing words

After solving, Ed says:

> Well you just made that look like a piece of cake! Though I prefer cookies...I know where to find the best in town!

Thanks again! See ya 'round!

----

## Extras

There's a lot to pick up on Ed's statements. Here I have tried to look into some of the most intriguing ones.

### Ed's wild history on getting an Enigma machine

I looked into this claim and found out Ed had documented it in a PDF located here: [Please Keep Your Brain Juice Off My Enigma](https://www.counterhack.net/Please%20Keep%20Your%20Brain%20Juice%20Off%20My%20Enigma%201.1SMALL.pdf)

The said Enigma machine:

![Ed Skoudis](/images/act1/act1-defang-extra-enigma.png)

The story follows cybersecurity professionals **Ed Skoudis** and **Josh Wright** as they embark on an unusual mission to purchase a genuine World War II Enigma machine. Ed first learned of a potential seller—**Dr. Tom**, a retired professor and Enigma enthusiast—through a broker named **Dr. David**. After earlier negotiations had stalled for unrelated personal reasons, Ed reconnected a year later and found that several Enigmas were available again.

Ed and Josh arranged a visit to Dr. Tom’s remote mountain farmhouse to personally inspect two machines. The trip required long drives for both of them, and the final stretch of road was steep and eerie, adding to the sense of adventure. Upon arrival, they were welcomed warmly by Dr. Tom and his wife, who gave them a detailed tour of the collection and took photographs throughout the visit.

Before making a purchase, Ed insisted on testing the Enigma’s functionality. Using an Enigma simulation app on his iPad, he encoded a message and verified that the real machine decoded it correctly. Satisfied with the test, he and Josh sat down to a surprisingly elegant lunch prepared by their hosts.

After reviewing the two available units—**A2200** and **A726**—they decided to purchase A726. The visit was filled with quirky moments, enthusiastic hospitality, and a sense of stepping into the home of someone deeply passionate about historical cryptography.

The transaction was completed successfully, and Ed and Josh left with a functioning Enigma machine and a story far more memorable than a simple equipment purchase. The narrative closes by reflecting on how the Enigma symbolizes both the deceptive confidence of flawed security and the ingenuity of those who unraveled it—while their journey highlights the unpredictable and strangely delightful experiences life offers.

### Ed's page 

It appears true that he owns a leaf of the Gutenberg Bible, according to this text found over at the [Cyphercon site](https://cyphercon.com/speaker/please-keep-your-bait-and-switch-business-practices-export-restrictions-and-customs-shipping-delays-off-my-incunable):

![Eds leaf of the Gutenberg Bible](/images/act1/act1-defang-extra-gutenberg-bible.png)

_These anecdotes adds historical and personal context to the challenge, but is intentionally separate from the technical objectives, serving as optional background rather than required analysis.__



