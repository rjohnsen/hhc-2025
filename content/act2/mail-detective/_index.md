+++
date = '2025-11-23T13:42:34+01:00'
draft = false
title = 'Mail Detective'
weight = 2
+++

![Maurice Wilson](/images/act2/mauricewilson-avatar.gif)

## Objective

Difficulty: 2/5

Help Mo in City Hall solve a curly email caper and crack the IMAP case. What is the URL of the pastebin service the gnomes are using?

## Maurice Wilson mission statement

> Hey there! I'm Mo, on loan from the Air Force, and let me tell you - Counter Hack is the best job I have ever had!
> 
> There is something in the works. Come see me later and we will talk. Can't say much right now, but check back later. You might be interested.
> 
> So here's our situation: those gnomes have been sending JavaScript-enabled emails to everyone in the neighborhood, and it's causing chaos.
> 
> We had to shut down all the email clients because they weren't blocking the malicious scripts - kind of like how we'd ground aircraft until we clear a security threat.
> 
> The only safe way to access the email server now is through curl - yes, the HTTP tool!
> 
> Think you can help me use curl to connect to the IMAP server and hunt down one of these gnome emails?

## Mail Detective: Curly IMAP Investigation

![Welcome screen](/images/act2/act2-mail-detective-1.png)

To solve this objective, we must use `curl` as a lightweight IMAP client. Because curl supports IMAP natively, we can authenticate and talk directly to the mail server. The first step is to list the available mailboxes:

```bash
curl "imap://localhost:143/" -u "dosismail:holidaymagic"
```

With no mailbox or message specified, curl sends an IMAP `LIST` command. This asks the server to return all mailboxes accessible to the user. The response looks like:

```bash
* LIST (\HasNoChildren) "." Spam
* LIST (\HasNoChildren) "." Sent
* LIST (\HasNoChildren) "." Archives
* LIST (\HasNoChildren) "." Drafts
* LIST (\HasNoChildren) "." INBOX
```

Each line describes a single mailbox. The IMAP attribute `\HasNoChildren` indicates that the folder has no subfolders, `"."` is the server’s hierarchy delimiter, and the final token is the mailbox name. This confirms that authentication succeeded and shows which folders we can query.

With the mailbox names in hand, we can start retrieving individual messages by their IMAP UID. For example:

```bash
curl "imap://localhost:143/INBOX;UID=1" -u "dosismail:holidaymagic"
```

fetches the message with UID 1 from `INBOX`. I iterated through all messages in `INBOX` but found nothing useful. I then repeated the process for the other folders. It wasn’t until I inspected the `Spam` folder that I found something interesting:

```bash
curl "imap://localhost:143/Spam;UID=2" -u "dosismail:holidaymagic"
```

The second message in the `Spam` folder contained the email I was looking for:

```html
Return-Path: <frozen.network@mysterymastermind.mail>
Delivered-To: dosis.residents@dosisneighborhood.mail
Received: from frost-command.mysterymastermind.mail (frost-command [10.0.0.15])
        by mail.dosisneighborhood.mail (Postfix) with ESMTP id GHI789
        for <dosis.residents@dosisneighborhood.mail>; Mon, 16 Sep 2025 12:10:00 +0000 (UTC)
From: "Frozen Network Bot" <frozen.network@mysterymastermind.mail>
To: "Dosis Neighborhood Residents" <dosis.residents@dosisneighborhood.mail>
Cc: "Jessica and Joshua" <siblings@dosisneighborhood.mail>, "CHI Team" <chi.team@counterhack.com>
Subject: Frost Protocol: Dosis Neighborhood Freezing Initiative
Date: Mon, 16 Sep 2025 12:10:00 +0000
Message-ID: <gnome-js-3@mysterymastermind.mail>
MIME-Version: 1.0
Content-Type: text/html; charset=UTF-8
Content-Transfer-Encoding: 7bit

<html>
<body>
<h1>Perpetual Winter Protocol Activated</h1>
<p>The mysterious mastermind's plan is proceeding... Dosis neighborhood will never thaw!</p>
<script>
function initCryptoMiner() {
    var worker = {
        start: function() {
            console.log("Frost's crypto miner started - mining FrostCoin for perpetual winter fund");
            this.interval = setInterval(function() {
                console.log("Mining FrostCoin... Hash rate: " + Math.floor(Math.random() * 1000) + " H/s");
            }, 5000);
        },
        stop: function() {
            clearInterval(this.interval);
        }
    };
    worker.start();
    return worker;
}

function exfiltrateData() {
    var sensitiveData = {
        hvacSystems: "Located " + Math.floor(Math.random() * 50) + " cooling units",
        thermostatData: "Temperature ranges: " + Math.floor(Math.random() * 30 + 60) + "°F",
        refrigerationUnits: "Found " + Math.floor(Math.random() * 20) + " commercial freezers",
        timestamp: new Date().toISOString()
    };
    
    console.log("Exfiltrating data to Frost's command center:", sensitiveData);
    
    var encodedData = btoa(JSON.stringify(sensitiveData));
    console.log("Encoded payload for Frost: " + encodedData.substr(0, 50) + "...");

    // pastebin exfiltration
    var pastebinUrl = "https://frostbin.atnas.mail/api/paste";
    var exfilPayload = {
        title: "HVAC_Survey_" + Date.now(),
        content: encodedData,
        expiration: "1W",
        private: "1",
        format: "json"
    };
    
    console.log("Sending stolen data to FrostBin pastebin service...");
    console.log("POST " + pastebinUrl);
    console.log("Payload: " + JSON.stringify(exfilPayload).substr(0, 100) + "...");
    console.log("Response: {\"id\":\"" + Math.random().toString(36).substr(2, 8) + "\",\"url\":\"https://frostbin.atnas.mail/raw/" + Math.random().toString(36).substr(2, 8) + "\"}");
}

function establishPersistence() {
    // Service worker registration
    if ('serviceWorker' in navigator) {
        console.log("Attempting to register Frost's persistent service worker...");
        console.log("Frost's persistence mechanism deployed");
    }
    
    localStorage.setItem("frost_persistence", JSON.stringify({
        installDate: new Date().toISOString(),
        version: "gnome_v2.0",
        mission: "perpetual_winter_protocol"
    }));
}

var miner = initCryptoMiner();
exfiltrateData();
establishPersistence();

document.title = "Frost's Gnome Network - Temperature Control";
alert("All cooling systems in Dosis neighborhood are now property of Frost!");
document.body.innerHTML += "<p style='color: cyan;'>❄️ FROST'S DOMAIN ❄️</p>";

// Cleanup after 30 seconds
setTimeout(function() {
    miner.stop();
    console.log("Frost's operations going dark... tracks covered");
}, 30000);
</script>
</body>
</html>
```

The URL of the pastebin services the gnomes are using is `https://frostbin.atnas.mail/api/paste`

## Maurice Wilson mission debrief

> Outstanding work! You've mastered using curl for IMAP - that's some serious command-line skills that would make any Air Force tech proud.
>
> Counter Hack really is the best job I have ever had, especially when we get to solve problems like this!