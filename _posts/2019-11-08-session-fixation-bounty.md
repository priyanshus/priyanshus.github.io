---
layout: post
title: How I fixed Session Cookie
date: '2019-11-08 18:27:31 +0530'
published: true
tags: [bugbounty]
---

Recently, In a bug bounty program I found an issue where I was able to fix my session cookie. This issue is known as Session Fixation, where attacker prepares a link to vulnerable website. The link contains a session id. Attacker shares this link to victim by any medium.

When victim opens the shared link and logs in to website for which attack is planned, the website does not issue a new session id and authenticates the victim.

Thereafter, Attacker can use the same session id to log in as victim.

![]({{site.baseurl}}/assets/img/posts/session_fixation_flow.png)


## POC:

1. Open your browser's development console and issue below command:

    ```document.cookie = "sessionid=IHaveSetItForYou";```

2. Login to vulnerable app.

3. Check the cookie, if your `sessionid` is still `IHaveSetItForYou` then you have succeeded with Session Fixation.

**For reporting this issue to vulnerable app, I was rewarded with XXX (3 digits) Euro**.

![]({{site.baseurl}}/assets/img/posts/session_image.jpeg)
