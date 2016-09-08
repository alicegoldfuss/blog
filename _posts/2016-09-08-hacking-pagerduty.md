---
layout: post
title: "Hacking PagerDuty"
description: "PagerDuty, brute force attacks, and math!"
tags: [technical, hacking, pagerduty, brute force, math]
modified: 2016-09-08
categories: 
---

Back in January I decided to toy around with some internal tooling, so our support staff could directly page specific engineering teams. To do this, I dug into PagerDuty’s REST API with the aim of making a simple UI in front of it.

That’s when I noticed something strange.

After a few minutes of staring at the screen, cogs slowly turning, I had my weekend planned.

<!-- more -->

## The Setup

Here is an example call from the [PagerDuty API docs](https://v2.developer.pagerduty.com/page/events-api-reference#!/Event/post_create_event_json){:target="_blank"}:

{% highlight curl %}
curl -X POST --header 'Content-Type: application/json' 
--header 'Accept: application/json' -d 
'{ "service_key": "e93facc04764012d7bfb002500d5d1a6", 
"incident_key": "srv01/HTTP", 
"event_type": "trigger", 
"description": "FAILURE for production/HTTP on machine srv01.acme.com", 
"client": "Sample Monitoring Service", 
"client_url": "https://monitoring.service.com", 
"details": 
{ "ping time": "1500ms", 
"load avg": 0.75 }, 
"contexts": [ ] }' 
'https://events.pagerduty.com/generic/2010-04-15/create_event.json'
{% endhighlight %}

See that field `service_key`? That takes a 32 character hex value, and it’s all you need to trigger an alert for a service. Not just one of your services. Any service.

No account ID, no user API key, no form of 2FA. Just a 32 character string.

Which means, if you can guess the string, you can trigger an alert for any PagerDuty customer.

Very, very unethical.

Also, very fun.

Now I just needed proof.

## The Execution

At first I tried to figure out the hashing algorithm. How did PagerDuty derive this 32 character string? Was it a mashup of the account and service ID? Did it have to do with the customer registration date?

I messed around with MD5 and SHA1, combining different seed values, with no such luck. These are known weak hash functions, and I didn’t expect to hit gold. Most likely whatever algorithm PagerDuty used included entropy I had no way of recreating. But it’s always good to try the doorknob before breaking off the hinges.

Now, onto my battering ram.

If I couldn’t reverse-engineer a key, I would need to generate all possibilities. Then, hypothetically, I could send triggers to all of them, and strike true for some percentage of attempts. This is called a **brute force attack**.

I wrote a Python script to iterate through all possibilities in the [a-f][0-9] range and pipe them to a text file. I knew there would be a lot of entries and I would get impatient waiting for the script to complete. However, I figured I would get far enough to get some reasonable matches. I wasn’t going to actually spam PagerDuty’s API, just prove that I could get a positive hit on one of the service keys in my account.

I let the script run for 80 minutes before killing it. The resulting text file was 43G in size.

The last generated entry? **aaaaaaaaaaaaaaaaaaaaaaaafc74a7b1**

## The Reality

So, I finally decided to do the math. Just how many results was I looking at here?

The 32 character string has values in range [a-f] and [0-9]. That’s 16 possible values for 32 spots.

16<sup>32</sup> = **3.4028237e+38**

A big number, sure, but not very meaningful. Let’s look at it this way:

If I generated 100 keys per second for 100,000 years, I would generate only **1.0790283e+24** results.

“But Alice,” you may say, “no key will have twenty a’s in it. You’re wasting a lot of compute time on unnecessary repeats.”

Very astute assessment! If I generated all possible service keys while only allowing each value to repeat three times, I would be generating about **883,413,014,065,530,374,354,632,704,000,000 entries**.

At a rate of 100 per second, that’s **280,128,429,117,684,669,696,421 years**.

Much more reasonable.

## The Possibility

So, can it be done? Obviously my weekend experiment was a failure. But could someone else, someone with more resources, hack PagerDuty?

The [Brute Force Calculator](http://calc.opensecurityresearch.com/){:target="_blank"} is a fun tool that can tell us just that. You give it your password length, keys per second, and charset, and it will tell you how long it takes to brute force the entire keyspace.

I fed it a length of 32 and my custom charset (abcdef1234567890) and chose the fastest calculation rate listed (929803 keys/sec).

The calculator spat out **12 septillion years**.

But, let’s stretch a bit further. In 2012 a [25-GPU cracking cluster](http://arstechnica.com/security/2012/12/25-gpu-cluster-cracks-every-standard-windows-password-in-6-hours/){:target="_blank"} was able to compute every possible 8 character password combination in 5.5 hours. Its 350 billion guesses-per-second speed, applied to our entire keyspace, would take about 30,829,380,202,302,897,683 years. Limiting each character to three repeats per key would take about **80,036,694,033,624 years**.

I’ll wait.

Even if someone did generate all possible service keys, the amount of time and effort needed to send triggers using all of them is practically not worth it. After all, PagerDuty has around 7,000 customers, a microscopic drop in the bucket of possible keys, even if every customer has 50,000 services in their account.

The answer is right in front of you, only it’s the size of a grain of sand, and you’re standing on the beach. Good luck.

## The Conclusion

So, can you hack PagerDuty? The best way would be to reverse-engineer the keys, if you can. Otherwise, you’re better off randomly unplugging rack cables and paging people that way.

For what it’s worth, I did reach out to PagerDuty in January and inform them of my weekend shenanigans. A security engineer told me that, while they know the 32 character string is secure, they would consider increasing the key length to prevent possible attacks.

Guess my next long weekend is planned.

