---
layout: post
title: A possibility of Account Takeover in Medium
ext-js: https://platform.twitter.com/widgets.js
tags: [Web Security, Oauth, Bug Bounty]
---

There are times when you discover something that is very common and ordinary which just blows your mind and you start thinking, “How come I didn’t knew this before!?”. I recently had that kind of a moment when I came to know that Twitter allows users to change their username. This immediately triggered a thought, “What if somewhere Twitter integration in a website is depending on Twitter username?”. I know this sounds crazy, but I thought this was worth a try.

## The Vulnerability

Long story short, turns out the login from Twitter on Medium depends on the Twitter username! Since Twitter allows users to change their username, if someone having his medium account connected to Twitter changes his username, we can change our Twitter username to his and by logging in via Twitter in Medium, we’ll log into their account.

Here’s how it would look like:

1. Login to Medium, go to Settings and link the account to Twitter:

![Image](/img/blog/2019/acc-takeover-medium/0-LOrCFhQF9-KPALPg.png){: .center-block :}
![Image](/img/blog/2019/acc-takeover-medium/0-OCJnf_yOk3svme5V.png){: .center-block :}
![Image](/img/blog/2019/acc-takeover-medium/0-r6wd_CctVQD46aEP.png){: .center-block :}

2. Now let’s say this Twitter user changed his username to something else. I as an attacker will change my Twitter username to **notsoshant**.

![Image](/img/blog/2019/acc-takeover-medium/0-NWC-EN-yYllWjkAn.png){: .center-block :}

3. Now go to Medium and choose the Sign In with Twitter option. By using attacker’s Twitter profile to log in, we can notice Medium actually logs in to victim’s Medium account.

![Image](/img/blog/2019/acc-takeover-medium/0-1U6PGDNhRDvDO-2k.png){: .center-block :}

As stupid this vulnerability may look like, this obviously shows implementing OAuth can go very wrong sometimes. I don’t prefer OAuth in general because if an OAuth provider gets compromised, all the connected accounts will get compromised automatically. A [recent Facebook vulnerability](https://newsroom.fb.com/news/2018/09/security-update/) allowed attackers to access token for millions of users. If you are logged in to any service through Facebook, that account was compromised too! So folks, always prefer a new account for each website and avoid OAuth! And experts recommend this too.

blockquote align="center" class="twitter-tweet">
    <p lang="en" dir="ltr">Yet another reason why password managers make so much sense: I *never* sign up to a service using social login, I create a new account, generate the password via <a href="https://twitter.com/1Password?ref_src=twsrc%5Etfw">@1Password</a> then store it there. Think of it as sand-boxing all your identities. <a href="https://t.co/MzdNzOJrPC">https://t.co/MzdNzOJrPC</a></p>
    &mdash; Troy Hunt (@troyhunt)
    <a href="https://twitter.com/troyhunt/status/1045854796471660550?ref_src=twsrc%5Etfw">September 29, 2018</a>
</blockquote>

## Vulnerability Disclosure Timeline

**September 13th, 2018:** Vulnerability disclosed to Medium  
**September 25th, 2018:** Vulnerability patched by Medium. No rewards or credits given.

Yeah, no rewards or credits because they concluded:

>“Report does not meet the criteria outlined in the Bug Bounty program ([https://medium.com/policy/mediums-bug-bounty-disclosure-program-34b1c80764c2](https://medium.com/policy/mediums-bug-bounty-disclosure-program-34b1c80764c2)), and therefore is not eligible for a reward.”

I guess it’s because it falls in the category of “Bugs that require unlikely user interaction or phishing”.
