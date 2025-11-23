+++
date = '2025-11-08T16:29:40+01:00'
draft = false
hidden = false
title = 'Visual Firewall Thinger'
weight = 7
+++

![Chris Elgee](/images/act1/chriselgee-avatar.gif)

## Objective

Difficulty: 1/5

Find Elgee in the big hotel for a firewall frolic and some techy fun.

## Chris Elgee mission statement

> Oh hi! Am I on the road again? I should buy souvenirs for the family.
> 
> Loud shirts? Love them. Because - hey, if you aren't having fun, what are you even doing??
> 
> And yes, finger guns are 100% appropriate for military portraits.
> 
> ... We should get dessert soon!
> 
> ----
> 
> Welcome to my little corner of network security! finger guns
> 
> I've whipped up something sweeter than my favorite whoopie pie - an interactive firewall simulator that'll teach you more in ten minutes than most textbooks do in ten chapters.
> 
> Don't worry about breaking anything; that's half the fun of learning!
> 
> Ready to dig in?

## Holiday Firewall Simulator

Upon entering the simulator we are presented with a fairly nice GUI. By spending a couple of minutes it is all quite forward what to do. It tells us exactly what to do under the "Firewall Configuration Goals" section. Let's get cracking!

![The simulator in all its glory](/images/act1/act1-firewall-1.png)

Clicking on the "Internet" node in the network drawing we have to enable `http` and `https` and click "Save Rules".

![Configuring the Internet](/images/act1/act1-firewall-2.png)

Next, we just tag along enabling `HTTPS`, `HTTP` and `SSH` for the DMZ:

![Configuring the DMZ](/images/act1/act1-firewall-3.png)

For the internal network to cloud we enable `http`, `https`, `smtp` and `ssh`:

![Configuring the internal network to cloud](/images/act1/act1-firewall-4.png)

We aren't quite finished with the internal network yet. We have to configure the connections to the workstations as well. These shall have full access.

![Configuring workstations](/images/act1/act1-firewall-5.png)

And Done! 

![Done](/images/act1/act1-firewall-6.png)

## Chris Elgee mission debrief

> finger guns Nice work! You've mastered those firewall fundamentals like a true network security pro.
>
> Now that was way more fun than sitting through another boring lecture, wasn't it?