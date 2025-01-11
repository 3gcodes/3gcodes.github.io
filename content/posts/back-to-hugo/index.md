+++
date = '2024-01-11T16:11:31-06:00'
draft = false
title = 'Back to Hugo for This Blog'
[cover]
    image = 'hugo-logo-wide.svg'
+++
The very first time I started blogging I used a Java based blogging platform called [Pebble](https://pebble.sourceforge.net/). If you look at it, you'll understand how long ago that was. Blogging had just become something people did and I had some level of success mostly due to the newness of it all. I was writing "how-to" articles and they seemed to resonate. At that time I was also heavily involved with [CodeRanch](https://www.coderanch.com), formally known as JavaRanch. So I would often take my answers there, and expound on them on my blog.

Over the course of time I've stopped and started blogging several times, eventually finding my way through Wordpress and then to static side generated platforms. My inconsistency over the last several years has been the meduim. Does anyone actually still want to consume the written format? Or is video media, like YouTube, the defacto.

I've come to a personal conviction now that I write better than I YouTube, if YouTube were a verb. As I started looking for the blogging platform I wanted to use I had a few requirements in mind:

- Dead simple to just write
- Light weight, fast
- Moderatly configurable
- Not Wordpress.
- Image support (you'd be surprised how elusive this is)

I had used [Hugo](https://gohugo.io/) in the past. And while it (mostly) met all my requirements, I found myself fiddling with it too much instead of writing. So this time around I decided to try [Write.as](https://write.as/). I was pleased with it. I published my first article a few days ago and got some good traction on it. However, as I started looking into some customizations and features that I wanted to (eventually) add, it began to lose it's appeal.

So today, as I'm writing this in a Markdown file, I have moved back to Hugo. At the end of the day, for what I need it's just the perfect solution. As a coder, I can pretty much make it do anything I need. As a blogger, I can ignore all that and just write.

I'm using the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme which has some nice features like read time, cover images, TOC among others. And I love how simple and clean the site is. I want the focus to be on the articles and not much else. Honestly, the hardest part about getting this up today was using my custom domain with Github pages. Here's a list of what I had to do:

- Install hugo (well, I just needed to update it)
- Install my theme (git submodule)
- Update the hugo config
- Added a hugo github action config
- Copied my existing Write.as article into the site
- Pushed up to my github pages repo.
- Update nameservers for my domain
- Tell github to use said domain

All in all, I think I spent a couple of hours getting things to where I felt like I could just write and not fiddle. Will I fiddle? Of course. But for now, I'm writing and I hope this inspires someone else to just write. In a world of short attention spans, YouTube, Twitch and TikTok (for now), all of which are terrific platforms, I think the art of writing is worth upholding and I not only want more peoole to spend time in the written format but I want to consume more written format. *Especially human written.*