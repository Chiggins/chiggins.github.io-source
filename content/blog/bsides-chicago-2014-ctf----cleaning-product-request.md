---
title: "Bsides Chicago 2014 CTF - Cleaning Product Request"
date: 2014-04-28T17:49:48-05:00
draft: false
---

At BSides Chicago 2014 this weekend I participated in the Tricity BSJTF CTF with team Penguins. One of the challenges that caused me the most rage and an epic face-palm once I figured it out was the “Cleaning Product Request” easy web challenge. Yes, I know, it was an “easy” challenge. I was just over thinking it and kept beating my head against the wall.

> BEGIN TRANSMISSION
>
> TARGET: BSides Joint Task Force LOCATION: TOP SECRET ()
>
> DETAILS:
>
> During our analysis of local agents systems we found a rogue file on our systems. We are not sure what it does but we are sure an agent has something to do with it. We tried to refer this to to our guys down in the lab but they said it should have been referred to someone else. That’s why we are getting you involved.
>
> https://bsjtf.com/f291lsf.php?flag=gimmie
>
> Hint: AJAX
>
> Good luck agent!
>
>END TRANSMISSION

So when you first visit the website the website, all you get is a message saying “Access Denied.” I started up Burp and captured the request to see if anything interesting stuck out, but no luck. I also tried to mess around with the referer header since that was the key to a challenge at last years CTF, but again no luck. The @bsjtf was tweeting out hints related to this challenge such as “What’s a utility that is also a cleaning product?”, “There are all sorts of things that can be “spoofed”. Especially when looking for “cleaning product” requests. #tricityctf”, and what finally gave me the final push, “How do you check to see if an request was sent via AJAX?”.

![Screenshot 1](/images/bsides_2014_cleaning/1.png)

If you look up how AJAX requests work, you’ll see that the `X_REQUESTED_WITH` header is set to `XMLHTTPRequest`. In Burp I send the request to repeater, added in the `X_REQUESTED_WITH` header, set it to `XMLHTTPRequest``, sent out the request, and boom, flag acquired. Ben0xA, you’re going to give me gray hair with simple silly challenges like this!

![Screenshot 2](/images/bsides_2014_cleaning/2.png)

![Screenshot 3](/images/bsides_2014_cleaning/3.png)
