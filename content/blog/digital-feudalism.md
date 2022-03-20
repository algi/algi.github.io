---
title: "Digital feudalism and freedom"
date: 2022-03-20T18:02:25Z
description: "Using plain text to regain my freedom back."
slug: "digital-feudalism"
tags: [ "vim", "writing" ]
categories: [ "FreeBSD" ]
---

I have recently read an interesting article about [why to write in a plain text](https://sive.rs/plaintext). What inspired me during reading it was fact that the author propagates writing files in plain text format. It also relates to my own path to freedom, which started by moving to FreeBSD.

# Rise of Digital Feudalism
One of the most prevalent form of paying for software is the subscription model, heavily pushed by many IT companies. While this model might allow sustainable source of income for the company, it removes an important aspect from the users - the ability to own the software or data. That means that users can only rent their OS, apps, their own data, music, family photos and many other things. As long as they pay and trust the company with their privacy, security and quality, they are fine.

My problem with this concept started with my iPad Air. At some point Apple has decided that this perfectly functional piece of hardware is no longer supported. At the beginning, it didn't mean much, just that the latest iOS wasn't available for it. But then slowly other apps started following the suite, including also security updates. At that point I realised that I was basically subscribing to Apple for hardware I was supposed to regularly buy in exchange for the possibility to access my own data.

One of the other examples of where I lost my ownership was my meal planner with a recipes database. I used Numbers for a very simple spreadsheet, where I put various recipes I collected over time and used for creating a weekly cooking plan. One of the few areas where iPad is convenient and useful is in the kitchen, where I could simply open the spreadsheet and click on the link with the recipe. That suddenly stopped working after an iOS update. My own data was no longer available to me and I had no other way than to use my Mac to access them. I can imagine that if I stopped getting new Macs, I would lose the access from my computer as well.

Such is the price for living in digital feudalism that we all pay for. As long as you pay your tithe collector for renting what used to be yours, you are good. The moment you stop, you loose the access immediately.

# Path to the Freedom
What is the solution for the problem then, I asked myself? How can I get the **ownership** of my hardware, OS, data and apps back? The answer is surprisingly simple and the solution completely free - to use open source, open formats and store data offline. By using open source OS like FreeBSD, I'm no longer forced to update my hardware every time some big tech CEO decides it's time to pay the price. I can even use a 10-year-old computer if I like. I'm still getting all the important apps I need - a modern browser, music player, text editor, etc. Also by storing my data offline, I can truly own it and control the access to it.

Getting my **data** out of cloud was easy, when it came to my photos and music. My personal notes were a different story though. Previously I used Notes app for writing and storing my ideas in iCloud. The solution for this approach was simple - I transferred them all to plain text files, written in Markdown. I use `vim` as my text editor of choice because it's convenient and the way it works almost hasn't changed since it was created. This makes the time that I spent mastering vim a life-long investment. Also, I organise my notes in folders. This doesn't require any proprietary technology and also allows me to use my muscle memory to remember where things are. I never had a problem of not being able to locate my notes. This can of course change if I start to have a massive amount of notes in the future. I have seen people being able to create simple Perl scripts to help them find what they need, so I'm not afraid that this problem would ever hit me hard.

The last advantage of my system is the possibility of having an offline backup. I use an external encrypted hard drive for it. Because I like to keep things simple, I use `rclone` as my backup solution. And that's all. This way I have achieved the following: privacy, security and freedom. What else has value in today's IT world?

---
*Version history of this blog post:*
1. *Very grumpy, complains about how big tech is evil and we are all victims of it.*
2. *Some basic ideas about digital feudalism. Mostly focused on saying that I use vim and I organise stuff in folders.*
3. *Focused on explaining concept of digital feudalism. Offline files in plain text as solution for privacy, security and freedom.*

