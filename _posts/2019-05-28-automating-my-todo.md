---
layout: post
title: "Automating My Todo with GitHub and Twilio"
description: "My personal weekly release."
tags: [technical, automation, github, twilio]
modified: 2019-5-27
categories: 
---

I’m a very organized person. I know this because I love lists.

I write lists everywhere. In my notebook, in my Notes app, on empty envelopes, receipts, paper bags. As a kid I would write them on my wrists (my mother *hated* that). Periodically I compile the lists together, figure out which items to do first, and scribble them out when complete. And I know the system is working because everything gets done, mostly, eventually, at some point.

It doesn’t work very well.

<!-- more -->

There are two main issues with my list system:

1. I don’t know what to do first
2. I never feel done

Since everything is written in the same list, everything feels like the same priority. And when I check something off, there are still twenty other things left on the page. So, I never get a break. I never get a weekend.

But, as an engineer, I get things done all the time! And I know exactly what needs doing. And I feel good at a job well done. And--

Oh.

Oh *no*.

Oh no no no.

I need to agile my life.

## Kanban 4 Lyfe

I’ve been using a new, automated system for a while now and it really hits the spot. It uses a kanban board and release cycles to highlight what I need to get done each week. It then wraps all my Done items up into a personal “weekly release” and sends it to me every Friday, so I feel accomplished and can relax. Then on Sunday I revisit my backlog and pick out what needs to be done in the upcoming sprint.

My new system is made up of four parts:

1. GitHub project board
2. Twilio account
3. Python script
4. Cron job 

By the end of this post, you will know exactly how to set this system up for yourself. I will give you everything you need.

## GitHub Project Board

For my kanban setup, I use a private [User-owned project board](https://help.github.com/en/articles/about-project-boards){:target="_blank"}, initialized as “basic kanban.” This gave me three columns: To do, In progress, and Done. I changed “To do” to “Backlog,” changed “In progress” to “Priority,” and added a “Releases” column, for archiving purposes.

<figure class="center">
<img src="/images/kanban.png" style="text-align: center;" alt="A personal project board with items relating to killing all humans and petting every cat in the world.">
<figcaption></figcaption>
</figure>

At this point, I must tell you that I am a GitHub employee. I use a GitHub project board because, well, I’m on GitHub all the time. I already had a login. I know the people who made it. It was right there. And it’s free.

But, you can use whatever kanban software you want for this, as long as it has an API. You definitely don’t need to use GitHub for this step.

## Twilio Account

Originally, I was going to email myself the releases, but it turns out you can’t send random emails to gmail without [disabling some security features](https://support.google.com/accounts/answer/6010255){:target="_blank"}. So, no to that.

I decided to do text messages instead, and [Twilio](https://www.twilio.com/){:target="_blank"} was the obvious choice.

I signed up for a free trial account, which comes with a free Twilio number. Each time Twilio sends a text, it costs a small fee. However, they give you some credit to start with, and take the cost from that. So, while it’s free to get started, I will eventually have to put money into Twilio (pls give me more credits, Twilio friends).

I used Twilio’s [Python helper library](https://www.twilio.com/docs/libraries/python){:target="_blank"} and basically copy/pasted it right into my script. Easy peasy.

## Python Script

The Python script ([get it here!](https://gist.github.com/alicegoldfuss/d9a4a8cce0b45e3d37060a07aec616dc){:target="_blank"}) is the real meat of this workflow. I wrote it in Python3 and it has five steps:

1. Gather all items from the Done column
2. Combine them into one bulleted list
3. Use that list to create a new card in the Releases column
4. Text that same bulleted list to my phone
5. Archive all cards in the Done column

To run this script, you will need to first setup your environment by installing the needed packages:

```
pip3 install requests twilio
```

You will also need to plug in several environment variables into the script file:

* `GH_TOKEN` Your auth token from [https://github.com/settings/tokens](https://github.com/settings/tokens){:target="_blank"}
* `TW_SID` Your Account SID from [https://twilio.com/console](https://twilio.com/console){:target="_blank"}
* `TW_TOKEN` Your Auth Token from [https://twilio.com/console](https://twilio.com/console){:target="_blank"}
* `DONE` Done column id
* `RELEASE` Release column id
* `TW_PHONE` Your Twilio account phone number
* `PHONE` Your phone number

To get your `DONE` and `RELEASE` column ids, you’ll first need to get your project id.

```
curl -H "Accept: application/vnd.github.inertia-preview+json" https://api.github.com/users/YOUR_GITHUB_USERNAME/projects?access_token=GH_TOKEN
```

It will be listed as `id` in the output.

Use that id to get your column ids:

```
curl -H "Accept: application/vnd.github.inertia-preview+json" https://api.github.com/projects/PROJECT_ID/columns?access_token=GH_TOKEN
```

Pick out the ids for your Done and Release columns and plug them into your script!

Give your script a test run and makes sure it works:

```
$ python3 script_name.py
```

Make sure you’re using Python3, have the right packages installed, and are connected to the Internet.

<figure class="center">
<img src="/images/twilio_text.jpg" style="text-align: center;width:300px" alt="A text message listing this week's Done tasks.">
<figcaption></figcaption>
</figure>

## Cron Job

I use a local cron job to run this script, every Friday, at 5pm.

You could use Jenkins! You could extract the API keys and make it more secure! I don’t. If someone gets ahold of my laptop, I will have more things to worry about than some API keys. I use a cron job because this workflow is supposed to make my life easier, and I’m not setting up Jenkins in the name of easy.

If you’ve never setup a cron job before, the two commands you need to know are

* `crontab -e` to edit a cron job
* `crontab -l` to list scheduled cron jobs

`crontab -e` will open a cron file in a command line editor, like `vi`. You enter the cron information for your job, save, then verify it took with `crontab -l`.

Here’s what my cron job looks like:

```
0 17 * * FRI cd /Users/nautalice/code && /Users/nautalice/Envs/release-script/bin/python release.py
```

* `0 17 * * FRI` runs it every Friday at 5pm
* `cd /Users/nautalice/code` navigates to my script directory
* `/Users/nautalice/Envs/release-script/bin/python release.py` runs my script

I use Python virtual environments, which is why I have the long `Envs` path to run my script. That uses my project environment (where `requests` and `twilio` are installed) to run my program!

## That’s it!

That’s my new system, and I like it a lot. It’s really rewarding to get a text message every week reminding me what I did, implicitly giving me permission to enjoy my weekend. 

If you enjoy presents, you could compile a list of things to treat yourself with, then hack an extension that sends you your weekly release plus a randomly-selected gift from your list. You could even add some logic around price tiers...complete 10 items, get a gift from price tier 1, etc.

I hope this post helps get you started on your own weekly release system! Happy scripting.
