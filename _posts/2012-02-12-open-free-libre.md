---
layout: post
title:  "On free, open and libre…"
date:   2012-02-12 00:00:00 +0100
categories: blog
---
I have undertaken a bit of an experiment in releasing PageTurner: it’s free as in speech but not as in beer.

I’d like to share my reasoning behind it and the results so far.

Currently there are 3 versions of PageTurner:

  1. OSS Version: Available for download on my site or by building the source yourself. Also downloadable through the excellent F-Droid app-store. Free both as in beer and speech.
  1. Ad-supported version: free as in beer and available in the Android Market
  1. Paid version: available in the Android Market for 50 eurocents, or 99 dollar-cents (I wouldn’t let me go lower).

Now, you might have noticed that though this app is completely Free (libre) and open source, it actually looks like every other commercial, closed-source app in the market with an ad-supported version and a paid “pro” version.

## What is free software?

Before I get into my reasoning for going with this pricing scheme, I’d like to take a little side-trip into my views on the benefits of free software.

Often people think it just means “software available at no cost”, leading to the distinction between free as in beer or free as in speech. In Dutch we translate it as “vrije” software, meaning it grants you (the user) freedom. But how does it do that?

  1. You (or someone you trust) can inspect the software by reading the source code to make sure it doesn’t do anything you don’t want it to. If the software sends your data to some marketing agency or tries to take over your machine, it will be there in the code.
  1. If you want, you can adapt the software to your needs and change its behaviour to better suit you.
  1. You can learn how the software works and use it as a basis for your own work. Some restrictions may apply here.
  1. You may share the software with your friends, both in the original version or with the changes you have made.

The first point might sound esotheric since you probably don’t want to go through the hassle of checking and building the software. You don’t always need to though. The F-Droid Market I mentioned earlier make it their business to check apps and build them from source so that they can be 100% sure that no nasty stuff creeps in there.

## OSS/Free Pricing

Now, as you might have noticed, none of the points about free software say anything about price. By now you’ll probably be saying: “yeah, true but every single open source app I know is downloadable free of charge!”, and you’d be absolutely right. The reason for that of course is mainly point 4: it doesn’t make much sense to charge people to get your software if they’re then allowed to give it away for free. In a sense you’d be competing with yourself.

## Applied to PageTurner

It is my personal belief that people aren’t unwilling to pay for things they like, as long as you make it easy and convenient for them. In fact, they’re generally willing to pay to save themselves some work. On the other hand there’s also a group of people that will gladly put in some effort if it saves them some money. I decided to try and accommodate both groups, while still honouring all the ideals behind free software and hopefully recouping some of the expenses I incur in running the backend-services for PageTurner.

So, how does it work? PageTurner really consists of 2 parts:

  * An Android reader app
  * A back-end server to store sync-points

Both of these components are Free with the source code available on my Github page. That means that if you want to you could easily:

  1. set up your own back-end server
  1. change 1 line of source code in the Android app
  1. Compile/package the android app and install it on your devices

You’d be up and running with 100% functionality without having to pay anyone, except perhaps for the server.

If however you’re like most people you will not want the hassle of setting up your own server. That’s why I also run a back-end server and my builds of the Android app point towards this server. If you get a build from the market, it will be able to connect to this server out of the box.

If you get the OSS version from any place, it will require an Access Key to connect to the server. Right now I’m giving anybody who contributes to the project in any way (donations, translations, testing) a free key. I might eventually set up a page where you can buy a key through PayPal in an automated way.

All 3 versions of the app are functionally identical. The only difference is that the OSS version has a preference where you can input your key, and this field is missing from the Ad-supported and Pro version. The keys are checked by the server against the keys in its database and depending on that your requests will be processed or refused.

I believe this gives users the full spectrum of freedom ranging from setting up everything yourself and keeping your wallet closed to having someone (me) take care of everything and paying for that either by watching ads or through a small one-time fee.

## The Big Question

To me this seemed like a good way to find a balance between generating some revenue from my app and honouring my belief in Free Software. I have seen very little apps try anything similar though, so I’m very curious to your opinion about my strategy.

So far responses have been positive, but I invite critical thinking. Just keep the language nice :) 
