+++
date = '2025-11-23T13:43:19+01:00'
draft = false
title = 'Quantognome Leap'
weight = 6
+++

![Charlie Goldner](/images/act2/charliegoldner-avatar.gif)

## Objective

Difficulty: 2/5

Charlie in the hotel has quantum gnome mysteries waiting to be solved. What is the flag that you find?

## Charlie Goldner mission statement

> Hello! I'm not JJ. I like music.
> 
> I accept AI tokens.
> 
> I like quantum pancakes.
> 
> I enjoy social engineeering.
>
> ----
> 
> Check back with me later - I might have something for you to help with!
>
> ----
> 
> I just spotted a mysterious gnome - he winked and vanished, or maybe he’s still here?
> 
> Things are getting strange, and I think we’ve wandered into a quantum conundrum!
> 
> If you help me unravel these riddles, we might just outsmart future quantum computers.
> 
> Cryptic puzzles, quirky gnomes, and post-quantum secrets—will you leap with me?

## Solution

![Welcome screen](/images/act2/act2-quantqnome-1.png)

Finding the PQC key generation program is quite easy, as we just can type `pq` and hit `TAB` and the auto completion will complete the rest for us: 

![Finding the key generator](/images/act2/act2-quantqnome-2.png)

Next, use -t to display key characteristics, as the screen instructs us to do:

![Finding the key generator](/images/act2/act2-quantqnome-3.png)

The hint states: "You can use 'ssh-keygen -l -f <private key>' to see the bit size of a key. Next step, SSH into pqc-server.com.". But first, lets see if there's a username I possibly can use:

![Finding SSH username](/images/act2/act2-quantqnome-4.png)

The username `gnome1` could possibly help me. Since the current user already has got a .SSH folder and keys, let's see if these leads anyway:

![Trying out SSH](/images/act2/act2-quantqnome-5.png)

And it does! 

![Logged in gnome1](/images/act2/act2-quantqnome-6.png)

So now we are trying to get access to user `gnome2`. Trying to see if we can rinse and repeat the process here:

![Finding gnome2 keys](/images/act2/act2-quantqnome-7.png)

Rinse and repeat worked:

![Logged in gnome2](/images/act2/act2-quantqnome-8.png)

Let me guess ... rinse and repeat for user `gnome3`?

![Finding gnome3 keys](/images/act2/act2-quantqnome-9.png)

Yes ... that worked ..

![Logged in gnome3](/images/act2/act2-quantqnome-10.png)

And for `gnome4`, we find this users keys as well ...

![Finding gnome4 keys](/images/act2/act2-quantqnome-11.png)

And we are in ...

![Logged in gnome4](/images/act2/act2-quantqnome-12.png)

And for `admin`, we find this users keys as well ...

![Finding admin keys](/images/act2/act2-quantqnome-13.png)

And we are in as admin...

![Logged in admin](/images/act2/act2-quantqnome-14.png)

Finding the flag:

![Finding the flag](/images/act2/act2-quantqnome-15.png)

Flag is `HHC{L3aping_0v3r_Quantum_Crypt0}`

## Charlie Goldner mission debrief

> That was wild—who knew quantum gnomes could hide so many secrets?
>
> Thanks for helping me leap into the future of cryptography!