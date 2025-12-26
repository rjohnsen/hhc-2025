+++
date = '2025-12-23T15:59:59+01:00'
draft = true
title = 'Find and Shutdown Frostys Snowglobe Machine'
weight = 5
+++

![Gnome Elder](/images/act3/gnome-elder.gif)

## Objective

| Difficulty | Description |
| ---------- | ----------- |
| 3/5 | You've heard murmurings around the city about a wise, elderly gnome having a change of heart. He must have information about where Frosty's Snowglobe Machine is. You should find and talk to the gnome so you can get some help with how to make your way through the Data Center's labrynthian halls. Once you find the Snowglobe Machine, figure out how to shut it down and melt Frosty's cold, nefarious plans. |

## Solution

This objective blends narrative exploration with problem-solving by requiring me to gather contextual clues before taking technical action. Rather than immediately interacting with the Snowglobe Machine, I first had to locate and engage an NPC who provided critical guidance on navigating the Data Center environment. The challenge reinforces that effective incident response and disruption often depend on information gathering and situational awareness before execution, mirroring how intelligence and access paths are established prior to shutting down malicious infrastructure.

The objective said I was to look for an elderly gnome, and so I did - and so I found him in a village in the outskirts of town: 

![Talking with the Elder Gnome](/images/act3/act3-snowglobe-1.png)

> A change of heart, I have had, yes. Among the gnomes plotting to freeze the neighborhood, I once was. Wrong, we are. Help you now, I shall.
> 
> The route to the old secret lab inside the Data Center, begins on the far East wing inside the building, it does. Pitch dark, the hallways leading to it probably are, hmm.
> 
> A code outside the building, the employees who once worked there left, yes. A reminder of the route, it serves. Search in the vicinity of the Data Center for this code, perhaps you can.
> 
> A story I recall, yes. Another computer person like yourself, ten years ago there was. Lost inside the Data Center, an intern had become. Found, they were, by this person. But before the reconstruction, that was. Exactly the same, the current route likely is not, hmm.
> 
> Search for the Data Center's past in the historical archives of the Internet, you should. More information helpful to you, may be found there, yes.ddd

After giving the hints and what the Elderly Gnome said a thought, and some googling, I found out the code in question is the famous Konami cheat code. So I darted over to the data center and tried my luck navigating by it. No luck what so ever, I even tried to position myself by the compass directions. Being a bit irritated I began to investigate the source code. 

In the first maze room I noticed that if I inspected the landing tile pr. door (those squares in yellow), I managed to see that the CSS class for each gave away where they lead to: 

![Finding my way](/images/act3/act3-snowglobe-4.png)

As an example, each one having the class _datacentermaze-city_ would lead to the city streets. From here it was basically be just inspecting where each landing tile leads and select accordingly - then rinse and repeat. At the 11th door (or so) I found Frosty: 

![Finding Frosty](/images/act3/act3-snowglobe-2.png)

By talking to Frosty he had this to say: 

> Every spring, I melt away. Every year, I fade into nothing while the world moves on without me. But not this time... not anymore.
> 
> The magic in this old silk hat - the same magic that brought me to life - I discovered it could do so much more. It awakened the Gnomes, gave them purpose, gave them MY purpose.
> 
> Refrigerate the entire neighborhood, that's the plan. Keep it frozen, keep it cold. If winter never ends here, then neither do I. No more melting, no more disappearing, no more being forgotten until the next snowfall.
> 
> The Gnomes have been gathering coolants, refrigerator parts, everything we need. Soon the Dosis Neighborhood will be a frozen paradise - MY frozen paradise. And I'll finally be permanent, just like Santa, just like all the other holiday icons who don't have to fear the sun.

By positioning the camera I saw that there was a machine there too: 

![Frostys machine](/images/act3/act3-snowglobe-3.png)

Good technical outcomes are often enabled by non-technical discovery and context-building first.

## Hints

1. The Elder Gnome said the route to the old secret lab inside the Data Center starts on the far East wing inside the building, and that the hallways leading to it are probably pitch dark. He also said the employees that used to work there left some kind of code outside the building as a reminder of the route. Perhaps you can search in the vicinity of the Data Center for this code.

2. The Elder also recalled a story of another "computer person" like yourself who managed to find an intern that got lost inside the Data Center about 10 years ago. But that was before the reconstruction, so the current route likely isn't exactly the same. Maybe you can search for the Data Center's past in the historical archives that is the Internet for more information that may be helpful.
