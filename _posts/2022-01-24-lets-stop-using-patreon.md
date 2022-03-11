---
layout: post
title: Stop Using Patreon, Substack, and Apple
author: Neal
categories: [media, patreon, wagtail, django]
image: /assets/img/posts/money.jpg
alt: money
intro: ...media distribution isn't worth paying 18% to 30% of your gross revenue.
---
<div class="row">
    <div class="col-12 col-md-4 my-auto">
        <h4 class="section-heading text-uppercase text-center">By {{ page.author }}</h4>
    </div>
    <div class="col-12 col-md-4 text-center team-member mt-4">
        <img class="img-fluid rounded-circle" src="{{ page.image }}" alt="{{ page.alt }}">
    </div>
    <div class="col-12 col-md-4 my-auto">
        <h4 class="section-heading text-uppercase text-center"> &bull; {{ page.date | date: "%b %-d, %Y" }}</h4>
    </div>
</div>

As I alluded to in a previous post, I have been writing an alternative for Patreon and its media distribution framework. This week it will be posted on Github and I've decided to make it truly free, open-source, no strings attached for content creators.

Here's why:

1. This project began as something for me to use myself, as I had a podcast idea that I've been working on, and after my co-host and I have gotten a few episodes in, we realize that we have to choose between the content and the software. If I were to try to sell this software we would be a software company, not content creators.  There isn't enough hours in a day for us to support this as a commercial product, so giving the software away to help promote our podcast makes more sense.

2. The only way regular Joes compete with finance-funded software projects is via collaboration in the form of open-source. We don't have billions of dollars (or even thousands of dollars) to throw at a software project. I have 20 years of experience with this stuff so writing the software I can do, but the real cost is in the maintenance and fine tuning of the finished result. Open source can do those things without money via crowd sourced collaboration, closed source software "products" need a budget for them. If other people use what I've built to host their own content, and submit ideas and changes back that they make to their own particular deployment, the project can grow and flourish. Otherwise we would have to seek financing to pay to move the ball forward and that would put us beholden to the same finance corporations we're trying to compete against.

3. For content itself to evolve, it needs to escape the platform mentality that constrains it currently.  As long as people are restricted to the mediums and media formats that giant tech corporations have pre-chosen for them, they will be subservient to those tech corporations, and those tech corporations will extract their fees at excessive rates from content creators just as print publishers have done to authors of written words for centuries. The solution to this problem is not to make another publisher, the solution is to *be your own publisher, which is to **not have one***.  This is, after all, the promise of the internet that at one time seemed fulfilled: "everyone can be their own publisher, you don't have to pitch a corporation to publish your work for you (and keep most of the money therefrom)."

# Relevance and Timing

All of this is prescient this week, because a trend is forming on Twitter regarding Patreon's apparent decision to dump the content of video creators on Vimeo, and Vimeo's subsequent demand for payment to cover bandwidth usage from those content creators.

<div class="svgcontainer">
<blockquote class="twitter-tweet" data-theme="dark"><p lang="en" dir="ltr">Wow, terrible business practice on the part of Patreon and Vimeo. (Patreon silently uses Vimeo to host videos uploaded to their site, Vimeo then takes down all of a creator&#39;s content because they exceeded some un-advertised limit and demands $8k a year) <a href="https://t.co/xsKON6VNe5">https://t.co/xsKON6VNe5</a></p>&mdash; jwkritchie (@jwkritchie) <a href="https://twitter.com/jwkritchie/status/1485064554090356737?ref_src=twsrc%5Etfw">January 23, 2022</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
&nbsp;&nbsp;&nbsp;
<blockquote class="twitter-tweet" data-theme="dark"><p lang="en" dir="ltr">Vimeo notified me and said I&#39;m in the top 1% of their high bandwidth consuming users. They&#39;re forcing me to upgrade my account from a $900/year Premium plan to a $3000/year Custom Plan. I have one week to comply or they will nuke my account lmfao<br><br>Dying platform continues to die</p>&mdash; hate5six (@hate5six) <a href="https://twitter.com/hate5six/status/1481500463317012482?ref_src=twsrc%5Etfw">January 13, 2022</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

</div>

To put this in context, Vimeo is demanding around 10x what this "service" is worth, roughly... let me explain.  If you want to host your own video content, you're looking at having to pay someone $0.01 per gigabyte for the bandwidth transfer.  Now, there is some value to be sure in the website and its UI, embedded player, and what not. However, in Patreon's case it doesn't seem like Vimeo is providing that, because Patreon users have to access the content they're paying for via Patreon's site.  So in this case Vimeo is just a video relay.

In the examples above, Vimeo is asking for $0.08 to $0.10 per gigabyte for bandwidth, basically, in telling a 30,000 GB per year user that they have to pay $3,000.00 for a year of service and a 100,000 GB per year user that they have to pay $8,000.00 for a year of service.

But there's a problem, it's worth a tenth of that.

[Digital Ocean](https://www.digitalocean.com/pricing){:target="_blank" rel="noopener"} will happily sell you all of the bandwidth egress you desire for $0.01 per GB. [Linode](https://www.linode.com/pricing/){:target="_blank" rel="noopener"} will match that rate, as will [Backblaze](https://www.backblaze.com/blog/transparency-in-cloud-storage-costs/){:target="_blank" rel="noopener"}. Why would anyone in their right mind pay Vimeo in this attempt at extortion?


# Because They Don't Know How

In a word, the promise of the early internet was lost.  Platforms have, like a parasite, latched themselves on to the publisher role that print publishers mentioned at the top of this post previously occupied.  Instead of pitching your book or your radio show to a publishing house or a FCC radio frequency license holder and letting them pay you scraps of profit from the fruits of your labor, you point and click your way to surrendering anywhere from 15% to 20% of your gross revenue to a "platform" that also hijacks your users and makes them their own, just for good measure.  People can't avoid using these tech platforms because hosting your own content takes time and skills that aren't (and should not be) required to produce audio-visual art.

* Sure you *can* host your own content but you need a web developer
* Sure you *can* sign up your own users but you need a web developer
* Sure you *can* process your own payments but you need a web developer

If you're doing all of this from scratch you're going to be reinventing all of the wheels those of us who are web developers have invented already, as well.  Every web developer in the world has already set up Stripe payments for a client, set up a way for them to embed their own audio and videos in their website, and set up a user signup system for them to have people sign up for notifications and marketing alerts.  It's old hat, been done a million times by a million different people. 

So I'm going to give everyone the tools out of the box to do what Patreon does, what Substack does, and what Vimeo does *for* Patreon all in one open source package, ready to deploy.

# Caveat Emptor

To be honest, this will still require you to bring your own art and your own HTML templates.  If you want Wix / Squarespace then they can, in fact, point and click you a website. But I presume if you've read this far you have already figured out that you can't be a subscriber-based audio / video production company on Wix. 

It will require you to sign up for your own hosting provider and own your own domain name.  You'll also have to sign up for an email relaying service so that your users get their invoices and signup confirmations.

You'll also have to do the deployment, but that can be simplified to automated Ansible scripts and guides that can be copied and pasted from.  After all, those of us who were working at dotcoms in 1999 learned this way too, so yes you can do it, it's not that hard.  

What is impossible, is pointing and clicking your way to being a professional artist via "platforms as a service" without the precarity of owing that platform your subjugation in exchange for your livelihood.  As long as your users are theirs and your payment processing account is theirs, ***they own you***.

I plan to release my code on Wednesday of this week.  All you'll need is a Stripe account, an email account, a web hosting account, and a domain name. Then you can be... well... not completely free but a lot more free than you were when you got a surprise $8,000.00 threat to delete your content from Patreon and Vimeo.