---
title: "How Our \"Chill\" CTF Turned Into an Uptime Arms Race"
date: 2025-11-30
authors: ["AbdulAzeez AbdulHakeem"]
tags: ["ctf", "attack-defense", "web-security", "xss", "api-hacking", "burpsuite", "red-team", "tokens", "uptime"]
cover_image: "https://cdn-images-1.medium.com/max/800/1*lsa7WKS9-tlLtth8DLfrPA.jpeg"
medium_url: "https://medium.com/@ithelance/how-our-chill-ctf-turned-into-an-uptime-arms-race-f3af76f6e82e"
---

# How Our "Chill" CTF Turned Into an Uptime Arms Race

**Role:** Red team and *accidental uptime engineer*

---

***This wasn't one of those CTFs where you're dying on every challenge from day one.***

We were a team of five (six teams in total). For most of the game we were sitting comfortably in the top three, and honestly, we'd already accepted that we'd probably finish in third place. Nothing crazy, no panic. All I had to do was keep the uptime steady and capture flags regularly, which we had automated anyway.

Then the last three days happened.

Teams from the bottom of the table suddenly started shooting up the leaderboard, and it wasn't because they were dropping insane attack points. Their uptime graphs just went mad. That was the first "wait, what's going on here?" moment.

**At that point I had two options:**

1. Complain and watch them overtake us, or
2. Figure out what they were doing and find my own way to ramp uptime.

I picked option 2. This is where it gets interesting.

<figure>
  <img src="https://cdn-images-1.medium.com/max/800/1*K8noyaUrevw97oozkM38xw.jpeg" alt="The base scoreboard prior to the last three days">
  <figcaption>Fig 1.0 — The base scoreboard prior to those "Three days"</figcaption>
</figure>

---

## Step 1 — Understanding the Game (Properly This Time)

From a neutral point of view, I checked out how the game was designed:

- Every team has a web app to keep online.
- Uptime = points.
- Other teams can attack you and gain attack points while reducing your defense.

So uptime was not just "nice to have." It was a weapon. We were doing okay with manual uptime, but clearly some teams had found a smarter way to play within (or around) those rules.

I decided I had to reverse-engineer whatever they were doing.

---

## Step 2 — You Can't Just Log In Like a Normal Human

There was another problem: **XSS**.

Many of the pages in the platform had an XSS vulnerability that people were already using to steal other teams' auth tokens. If you just casually logged into your dashboard and moved around the site, your token could get grabbed and other teams could start using it to hit your endpoints.

If that happened, you'd basically be helping them farm activity with your own identity. So the rule for survival was:

**Do not log into the app normally.**

No "login → attack page → click around." If you did that, you were food.

The workaround was:

1. Use BurpSuite earlier in the game to grab a valid Authorization token for our team.
2. Stay logged out in the browser.
3. Manually load that token into localStorage via DevTools on only the page we cared about: the attack/uptime page, which did not have the vulnerability (or so I thought).

So you're basically on the attack page, logged out. Open DevTools → Application → Local Storage → paste in the token BurpSuite gave you.

Refresh.

Now the page thinks "oh, this is a legit team session," but your token never touched the risky pages.

A little ghetto, but it worked.

<figure>
  <img src="https://cdn-images-1.medium.com/max/800/1*lsa7WKS9-tlLtth8DLfrPA.jpeg" alt="The attack page">
  <figcaption>Fig 2.0 — The everly talked about "Attack page"</figcaption>
</figure>

<figure>
  <img src="https://cdn-images-1.medium.com/max/800/1*Naqy26A21QjjFBtyv5tjxw.jpeg" alt="Consequences of having your token in the open">
  <figcaption>Fig 2.1 — The name changes you see are but some of the consequences of having your token in the open</figcaption>
</figure>

---

## Step 3 — Hunting for the Uptime API

Once I had a safe way to be "logged in" only on the attack page, I started doing the boring but necessary part:

**Inspect. Every. Single. Request.**

Network tab open. Click everything on the attack page. Filter by fetch / xhr. Stare at URLs and payloads.

Eventually I found the endpoint that looked like it was tied to uptime.

On the backend side, there was an API that looked like this in the browser:

*(screenshot: Django REST Framework "Retrieve Recent Flag" view with `/api/challenges/1/latest-flag/?groupName=…`)*

The response looked something like:

```json
{
  "Cached": false,
  "Flag": "CSOC-{2DCYB3RW4V-7ddfa926ba0556c8-195002}"
}
```

<figure>
  <img src="https://cdn-images-1.medium.com/max/800/1*rtqzBV_KGxDDLYlBU77F7w.jpeg" alt="DevTools Network Preview tab showing the flag response">
  <figcaption>Fig 3.0 — DevTools → Network → Preview tab showing Cached: false, Flag: "CSOC-{…}"</figcaption>
</figure>

That `Flag` value was what I'd later call the "uptime flag."

The interesting part: the request body and headers showed that the endpoint was happy to accept a JSON payload, and it didn't need any fancy browser-only behaviour beyond the usual auth headers.

So now I knew:

- Which endpoint was tied to uptime.
- What the expected JSON format looked like.
- That my manually injected token worked there.

Time to poke it directly.

---

## Step 4 — Curling the Payload Manually

I copied the request as curl from DevTools, swapped in my token and the JSON body, and fired it from my terminal. Normally, uptime would tick slowly — like 0.5 every 10 seconds if the service stayed online. After my manual curl request, I refreshed the scoreboard. Our uptime jumped immediately.

I tried it again.

It jumped again… just about time for the moderator to enforce that there's a heavy focus on using a real browser and even used **JA4 TLS fingerprinting** to block bots and scripts so people wouldn't just farm uptime with curl or Python. He thought he did something, 😆.

Two bumps in under two seconds. No waiting. No slow natural increase. That was the "ohhh okay, I'm onto something" moment. At that point I knew two things:

1. This endpoint was definitely wired into uptime.
2. It wasn't fully constrained to "browser only" the way the organiser probably intended.

---

## Step 5 — Turning It Into an Uptime Machine

Once I knew it worked, the rest was straightforward:

1. Clean up the curl request.
2. Wrap it inside a small script (bash/Python, whatever you like).
3. Add a loop with a sensible interval so I wouldn't accidentally DOS anything or draw too much attention.
4. Let it run and watch the scoreboard.

So instead of:
- Waiting for natural uptime
- or spamming buttons manually in a browser

I now had a way to send valid uptime-boosting requests on repeat.

Those last three days turned into a proper race. We went from "comfortably third" to actually fighting for position against teams that had looked dead earlier in the game.

---

## Flags, APIs and All the Side Quests

While uptime was the big play near the end, this CTF wasn't just about that one trick.

Along the way I also:

- Pulled flags from other APIs by inspecting the request flows.
- Used ChatGPT like a rubber duck, dumping ideas and frustration every time I got stuck.
- Learned how ugly XSS + stolen tokens can be in a real attack-and-defense setup — your bank apps better be ready for me 🤓.
- Got more comfortable thinking in terms of "what is the frontend hiding from me?" instead of just clicking around like a normal user.

And when I later read the organizer's blog post about using JA4 TLS fingerprinting to block bots and scripts from farming uptime, it made the whole thing even more interesting.

They were actively trying to keep everyone inside the browser. Meanwhile I was living half in the browser, half in DevTools, half in Burp, half in curl.

Yes, that's four halves. That's what the last few days felt like.

---

## What I'm Taking Away From This CTF

No motivational speech, just straight lessons:

- **Don't trust the UI.** If there's an API behind it, look at the API.
- **Tokens (or their equivalent) are everything.** Learn how they move, where they're stored, and how they can be stolen.
- **Attack pages can be safer than dashboards** if you understand where the real vulnerabilities live.
- **Uptime can be a bigger weapon than pure attack points** in an attack-defense CTF.
- And sometimes, the win is not "I found a crazy RCE," it's "I understood the game mechanics better than yesterday."

---

That's the story.

Not polished, not PR-ready. Just how it actually played out: a chill CTF that turned wild in the last three days, and how digging into APIs, tokens and one JSON payload changed our position on the board.

**FROM 3rd TO 4th 😝!**
