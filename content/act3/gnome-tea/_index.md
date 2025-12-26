+++
date = '2025-12-23T15:59:59+01:00'
draft = true
title = 'Gnome Tea'
weight = 1
+++

![Thomas Bouve](/images/act3/thomasbouve-avatar.gif)

## Objective

| Difficulty | Description |
| ---------- | ----------- |
| 3/5 | Enter the apartment building near 24-7 and help Thomas infiltrate the GnomeTea social network and discover the secret agent passphrase. |

## Thomas Bouve mission statement

> Hi there, I'm Thomas, but you can call me CraHan if you like.
>
> What do you mean 'weird accent'? I have no idea why people keep telling me that, but it sounds fine in my head, to be honest.
> 
> Anyway, way back before I joined Counter Hack, I was an avid HHC player just like you! Some of [my write-ups](https://n00.be/) even resulted in a couple of wins throughout the years. Definitely check them out if you're looking for some inspiration.
> 
> If you decide to submit your own report, my [Holiday Hack Challenge report template](https://github.com/crahan/HolidayHackChallengeTemplate) might help you save some time as well.
> 
> My t-shirt? You like it? Well, Synthwave, cyberpunk, and even cassette futurism are definitely my kinda jam!
> 
> I also love to tinker, but for some weird reason my drawer of unfinished hacking projects just keeps overflowing. 24 hours in a day simply isn't enough time, I guess.
> 
> Oh... and no matter what Mark or Kevin try to tell you, the Amiga is the absolute best retro computing platform ever made!
> 
> ----
>
> Hi again. Say, you wouldn't happen to have time to help me out with something?
> 
> The gnomes have been oddly suspicious and whispering to each other. In fact, I could've sworn I heard them use some sort of secret phrase. When I laughed right next to one, it said "passphrase denied". I asked what that was all about but it just giggled and ran away.
> 
> I know they've been using GnomeTea to "spill the tea" on one another, but I can't sign up 'cause I'm obviously not a gnome. I could sure use your expertise to infiltrate this app and figure out what their secret passphrase is.
> 
> I've tried a few things already, but as usual the whole... Uh, what's the word I'm looking for here? Oh right, "endeavor", ended up with the rest of my unfinished projects.

## Solution

This objective focuses on information disclosure through exposed application content and poor separation between operational data and presentation. By passively reviewing page structure and embedded content, I was able to extract a shared secret intended for internal use, demonstrating how sensitive operational details can be leaked without any active exploitation. The challenge reinforces that reconnaissance and careful reading often yield results before any technical attack is required.

Upon entering this mission we are presented with a [website](https://gnometea.web.app/login):

![Site landing page](/images/act3/act3-gnometea-1.png)

We have got a login form - usually in CTF's these sometime reveal relevant error messages. Let's see if we can provoke one by simply logging in with some random value: 

![Firebase error message](/images/act3/act3-gnometea-2.png)

Sure it does, of course it does. Somehow there is an `Firebase` data base lurking in the backend. Since Firebase is more than often coupled with Javascript, we look for some hints in the HTML source for some references. By inspecting the HTML source I see one interesting Javascript called `https://gnometea.web.app/assets/index-BVLyJWJ_.js`, and it does contain references to `Firebase`:

![Firebase references in Javascript](/images/act3/act3-gnometea-3.png)

### Investigation using Chat GPT

The Javascript is heavily minified and difficult to read. Since we live in the AI era, I uploaded the Javascript to ChatGPT and issued the following prompt: 

![ChatGPT to the rescue](/images/act3/act3-gnometea-4.png)

And bingo! We have extracted gold!

#### ChatGPT output

The file contains references to multiple Firestore collections and a Firebase Storage bucket. These were extracted from calls such as:

* `hs(er, "collectionName")`
* `Ro(hs(er, "collectionName"))`
* `Yf(er, "collectionName", ...)`

---

##### 1. Firestore Collections (Name + Description)

| Collection        | Description                                                                                                                                                      |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **gnomes**        | Used in the app to store metadata about gnome entities, likely linked to profile avatars or characters. Extracted from repeated references to `hs(er,"gnomes")`. |
| **tea**           | Appears to store "tea posts" or messages shown in the “tea feed”. Used in UI rendering.                                                                          |
| **dms**           | Direct messages between users. Extracted from `hs(er,"dms")` being passed into message components.                                                               |
| **admins**        | Holds admin user records or permissions. Access checks reference this collection.                                                                                |
| **users**         | Primary user profiles (names, avatars, IDs, roles). Extensively referenced throughout the code.                                                                  |
| **notifications** | Stores notification events for users. Seen in UI for listing alerts.                                                                                             |

These six collections are the only ones referenced in the JS bundle.

---

##### 2. Endpoints to View Each Collection

Using your Firebase project ID discovered (`holidayhack2025`), the Firestore REST API allows direct unauthenticated GET requests if the database rules are permissive.

Template:

```
https://firestore.googleapis.com/v1/projects/holidayhack2025/databases/(default)/documents/<collection>
```

Here are all collections with ready-to-use URLs:

| Collection    | URL                                                                                                        |
| ------------- | ---------------------------------------------------------------------------------------------------------- |
| gnomes        | `https://firestore.googleapis.com/v1/projects/holidayhack2025/databases/(default)/documents/gnomes`        |
| tea           | `https://firestore.googleapis.com/v1/projects/holidayhack2025/databases/(default)/documents/tea`           |
| dms           | `https://firestore.googleapis.com/v1/projects/holidayhack2025/databases/(default)/documents/dms`           |
| admins        | `https://firestore.googleapis.com/v1/projects/holidayhack2025/databases/(default)/documents/admins`        |
| users         | `https://firestore.googleapis.com/v1/projects/holidayhack2025/databases/(default)/documents/users`         |
| notifications | `https://firestore.googleapis.com/v1/projects/holidayhack2025/databases/(default)/documents/notifications` |

To list *all root-level collections*:

```
https://firestore.googleapis.com/v1/projects/holidayhack2025/databases/(default)/documents
```

---

##### 3. Endpoints to View Firebase Storage Bucket Contents

The JS file shows the bucket:

```
holidayhack2025.firebasestorage.app
```

Firebase Storage REST API uses the following format:

#### List all items in bucket

```
https://firebasestorage.googleapis.com/v0/b/holidayhack2025.firebasestorage.app/o
```

#### List items under a specific folder

```
https://firebasestorage.googleapis.com/v0/b/holidayhack2025.firebasestorage.app/o?prefix=<folder>/
```

#### Download a file (if public)

```
https://firebasestorage.googleapis.com/v0/b/holidayhack2025.firebasestorage.app/o/<path>?alt=media
```

#### Get metadata for a file

```
https://firebasestorage.googleapis.com/v0/b/holidayhack2025.firebasestorage.app/o/<path>
```

### Back to manual investigation

### Manual inspection

As we can see from the previous section, ChatGPT is quite capable of extracting interesting information from obfuscated information. However, as a side note, the HTML code of the landingpage also gives away hints on collection:

```html
<!-- TODO: lock down dms, tea, gnomes collections -->
```

There are many interesting collections to look at it turns out. However, `dms` caught my interest the most. 

```
https://firestore.googleapis.com/v1/projects/holidayhack2025/databases/(default)/documents/dms
```

By reading through the various messages passing back and forth I found out that a gnome named `Barnaby Briefcase` has forgotten his password. He reached out to `Glitch Mitnick` to reset it. Barnaby is reluctant to give out his password, but he gives out a hint: 

> Sorry, I can't give you my password but I can give you a hint. My password is actually the name of my hometown that I grew up in. I actually just visited there back when I signed up with my id to GnomeTea (I took my picture of my id there)."

The full conversation: 

![Password conversation](/images/act3/act3-gnometea-6.png)

That hint sounds interesting. From the looks of it, his ID is a file somewhere. Most likely somewhere in this bucket (which I found as mentioned above using ChatGPT):

```
https://firebasestorage.googleapis.com/v0/b/holidayhack2025.firebasestorage.app/o
```

A simple look into it reveals a lot of files and they are organized in a non easy searchable way. Luckily we have found an endpoint that list out every gnome in much simpler way: 

```
https://firestore.googleapis.com/v1/projects/holidayhack2025/databases/(default)/documents/gnomes
```

We search for `Barnaby` and finds his record:

![Barnabys record](/images/act3/act3-gnometea-7.png)

From the screenshot above we se an URL for the drivers license. However, that link doesn't work. We have to rewrite it slighty to: 

```
https://firebasestorage.googleapis.com/v0/b/holidayhack2025.firebasestorage.app/o/gnome-documents%2Fl7VS01K9GKV5ir5S8suDcwOFEpp2_drivers_license.jpeg?alt=media
```

Then we get the drivers license: 

![Barnabys driver license](/images/act3/l7VS01K9GKV5ir5S8suDcwOFEpp2_drivers_license.jpeg)

By looking at the meta data of the drivers license, we get a GPS coordinate. Here I have used [Pic2Map](https://www.pic2map.com):

![GPS cordinates from drivers licence](/images/act3/act3-gnometea-8.png)

By the looks of it Barnabys hometown is `Gnomesville`. That is supposedly the password we're looking for. In order to find Barnabys email we need to look it up in `https://firestore.googleapis.com/v1/projects/holidayhack2025/databases/(default)/documents/gnomes`:

```
barnabybriefcase@gnomemail.dosis
```

Using that email and that password (in lowercase) we were able to log in:

![Logging into GnomeTea](/images/act3/act3-gnometea-9.png)

ChatGPT mentioned something about an "admins" collection. If this exists, then surely there must be an `/admin` section to this site. Visting `https://gnometea.web.app/admin`, we see the following error: 

It complains that `window.ADMIN_UID: not set` is not set, so we set it and see what happens:

![Becoming admin](/images/act3/act3-gnometea-11.png)

And boom! We're admin:

![We are admin](/images/act3/act3-gnometea-12.png)

By scrolling down we find the secret pass phrase: `GigGigglesGiggler`

If secrets are rendered to the client, they should be considered compromised.

## Thomas Bouve closing words

After solving, Thomas says:

> Excellent! Now we can communicate with the gnomes. When I tried to talk to one just now it said "passphrase accepted".
> 
> I asked what they were up to and it said something about going to the old warehouse/data center at the appointed time for the next meeting. No clue what that means though.
> 
> Anyhoo, that's a pretty big item you helped remove from my pile of unfinished hacking projects. I really appreciate the assist!