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

As I were looking around the site I kept a close eye on the _/status_report_ endpoint, even if I kept myself to _/register_ I ended up with violations no matter what I did. Also, I noticed that there was a small gnome hiding in the bottom left corner when I looked over my screenshots. 

![Violations](/images/act3/act3-schrodinger-4.png)

### Getting rid of the vandaling Gnome

Reading up on the hints I was told to watch out for tiny, pesky gnomes who may be violating my progess. That gnome in the left bottom corner seemed to fit the bill. I then started to backtrack and reading up on the HTML source, and soon I found something interesting:

```javascript
<script>
(function() {
    if (window.gnomeVandalLoaded) return;
    window.gnomeVandalLoaded = true;
    
    document.addEventListener('DOMContentLoaded', function() {
        function getCookie(name) {
            var value = "; " + document.cookie;
            var parts = value.split("; " + name + "=");
            if (parts.length == 2) return parts.pop().split(";").shift();
        }

        if (getCookie('Schrodinger')) {
            var container = document.querySelector('.mini-gnome-container');
            if (container && !document.getElementById('mini-gnome')) {
                var img = document.createElement('img');
                img.src = '/gnomeU?id=b137f07d-7fd1-4537-8583-5ecd116a23f3';
                img.id = 'mini-gnome';
                img.style.cssText = 'width: 20px;height: auto;position: fixed; left: 10px; bottom: 10px;';
                container.appendChild(img);
            }
        }
    });
})();
```

This script seems available on all pagens and runs once per page load and checks whether a cookie named Schrodinger is present in the browser. If the cookie exists, the script dynamically injects a small image element into the page: a "mini gnome" anchored to the bottom-left corner of the viewport. A global guard (window.gnomeVandalLoaded) ensures the script does not execute more than once, even if the page logic attempts to load it repeatedly. In practice, this means that simply having the Schrodinger cookie set is enough to trigger the gnome’s appearance, regardless of which endpoint I was actively interacting with. I figured out this piece of code was responsible for the violations by adding a _HTTP match and replace rules_ rule tem to BurpSuite Proxy:

A simple rule set on the response body seemed to do the trick: 

| Type | Match | Replace | Where |
| ---- | ----- | ------- | ----- |
| Response body | `img\.src\s=\s'\/gnomeU\?` | `img.src = '/b0rk?` | To get rid of Gnome |
| Request header | `Priority: u=0, i` | `X-Forwarded-For: 127.0.0.1` | Invalid Forwarding IP |

By blocking the image source the violations seemed gone!

### Uncovered developer information disclosure.

Along the way one of the gnomes provides us with a sitemap. The list is quite long and I tried to see if any of the pages were reachable from the _/register_ path: 

| URL                                                                                    | Mapped URL | Comment |
| -------------------------------------------------------------------------------------- | ---------- | ------- |
| flask-schrodingers-scope-firestore.holidayhackchallenge.com/register                   |            | No response received |
| flask-schrodingers-scope-firestore.holidayhackchallenge.com/register/login             |            | Reachable |
| flask-schrodingers-scope-firestore.holidayhackchallenge.com/register/reset             |            | Reachable |
| flask-schrodingers-scope-firestore.holidayhackchallenge.com/register/sitemap           |            | Reachable |
| flask-schrodingers-scope-firestore.holidayhackchallenge.com/register/status_report     |            | Reachable |
| flask-schrodingers-scope-firestore.holidayhackchallenge.com/wip/register/dev/dev_notes | /register/dev/dev_notes | HTTP 403 error |
| flask-schrodingers-scope-firestore.holidayhackchallenge.com/wip/register/dev/dev_todos | /register/dev/dev_todos | Reachable page with vulnerability  |

 Path _/register/dev/dev\_todos_ contains _Uncovered developer information disclosure_:

 ![Uncovered developer information disclosure.](/images/act3/act3-schrodinger-5.png)

### Exploited Information Disclosure via login

Logged in using the credentials I just found: `teststudent:2025h0L1d4y5`

 ![Logging in as test student.](/images/act3/act3-schrodinger-6.png)

### Found commented-out course search

By inspecting the HTML source code I found an interesting comment mentioning a new endpoint: 

```html
<section class="courses-content">
    <div class="courses-container" style="max-width: 500px;">
        <h2 class="semester-heading">❄️ Spring Semester 2026 ❄️</h2>
        <!-- Should provide course listing here eventually instead of the extra step through search flow. -->
        <!-- <ul id="courseSearch" class="courses-list">
            <li><a href="/register/courses/search?id=1da0186b-4709-4edc-80be-fe43675c480d">Course Search</a></li>
        </ul> -->
    </div>
</section>
```

By removing the comment and thus enabling the list, we got: 

 ![Enabled course search](/images/act3/act3-schrodinger-7.png)

### Identified SQL injection vulnerability

By visting the link we just enabled I was presented with a search interface to search for courses. After toying with the query input, I figured out that this was SQL Injection all day long: 

 ![SQL Injection](/images/act3/act3-schrodinger-8.png)

### Reported the unauthorized gnome course

At the end of the course list there's an entry named _"GNOME 827 - Mischief Management"_ - clicking on that link lead me to: 

 ![Report In-Scope vulnerability](/images/act3/act3-schrodinger-9.png)

The popup offers three options:

* Report 
* Remove Course
* Continue

At the same time it urges for me to report the vulnerability. And so I did. 



---


After finding the information disclosure, the onscreen gnomes hinted at this URL: `https://flask-schrodingers-scope-firestore.holidayhackchallenge.com/search/student_lookup?id=1da0186b-4709-4edc-80be-fe43675c480d`

However, this is out of scope. 



### XSS

### Hints

1. Though it might be more interesting to start off trying clever techniques and exploits, always start with the simple stuff first, such as reviewing HTML source code and basic SQLi.

2. Watch out for tiny, pesky gnomes who may be violating your progess. If you find one, figure out how they are getting into things and consider matching and replacing them out of your way.

3. During any kind of penetration test, always be on the lookout for items which may be predictable from the available information, such as application endpoints. Things like a sitemap can be helpful, even if it is old or incomplete. Other predictable values to look for are things like token and cookie values

4. Pay close attention to the instructions and be very wary of advice from the tongues of gnomes! Perhaps not ignore everything, but be careful!

5. As you test this with a tool like Burp Suite, resist temptations and stay true to the instructed path.


## Kevin McFarland mission debrief

> 