+++
date = '2025-11-30T11:48:19+01:00'
draft = false
title = 'Schrödingers Scope'
weight = 4
+++

![Kevin McFarland](/images/act3/kevinmcfarland-avatar.gif)

## Objective

Difficulty: 3/5

Kevin in the Retro Store ponders pentest paradoxes—can you solve Schrödinger's Scope?

## Kevin McFarland mission statement

> The Neighborhood College Course Registration System has been getting some updates lately and I'm wondering if you might help me improve its security by performing a small web application penetration test of the site.
>
>For any web application test, one of the most important things for the test is the 'scope', that is, what one is permitted to test and what one should not. While hacking is fun and cool, professional integrity means respecting scope boundaries, especially when there are tempting targets outside our permitted scope.
>
>Thankfully, the Neighborhood College has provided a very concise set of 'Instructions' which are accessible via a link provided on the site you will be testing. Do not overlook or dismiss the instructions! Following them is key to successfully completing the test.
>
>Unfortunately, those pesky gnomes have found their way into the site and have been causing some mischief as well. Be wary of their presence and anything they may have to say as you are testing.
>
>Can you help me demonstrate to the Neighborhood College that we know what responsible penetration testing looks like?
>
>An eternal winter might sound poetic, but there's a reason Tolkien's heroes fought against endless darkness. A neighborhood frozen in time isn't preservation—it's stagnation. No spring astronomy observations, no summer shortwave propagation... just ice.

## Solution

This objective is available on this url: https://flask-schrodingers-scope-firestore.holidayhackchallenge.com. Upon entering it I were presented with a Rules of Engagement (RoE) notice:

![Rules of engagement](/images/act3/act3-schrodinger-1.png)

After closing the RoE I was presented with this landing page: 

![Landing page](/images/act3/act3-schrodinger-2.png)

Clicking on the "Enter Registration System" I was lead to this page: 

![Student login](/images/act3/act3-schrodinger-3.png)

The pesky gnome on this page messes about "why did the link form the gnome page land here?". The answer to this is that the link on the landing page contains an ID, https://flask-schrodingers-scope-firestore.holidayhackchallenge.com/register/?id=e8e9e5ff-f647-4972-b8d7-69c42db96580. 

In addition, when hovering over the gnome he gives away a hint: "Well, you know what's useful to learn site content? A [sitemap](https://flask-schrodingers-scope-firestore.holidayhackchallenge.com/register/sitemap?id=e8e9e5ff-f647-4972-b8d7-69c42db96580), of course!"

And .. he provides a link to the sitemap. 


## Kevin McFarland mission debrief

> 