---
layout: post
title: "Introducing: Sarlacc - an SMTP Sinkhole"
---

![sarlacc database](/images/sarlacc-database.png "sarlacc database")

A few months ago I published a [blog post](/chasing-necurs/) on my necurs tracker and in it I described my spam collection setup in my lab. My setup is still very much the same except for one thing: the smtp server. Previously I was using [mailslurper](https://github.com/mailslurper/mailslurper) but found that it was inadequite for my purposes. This was mainly because of the way the database was structured, every mail item that mailslurper collects gets stored with no relational structure so you end up with millions of copies of the same data data being stored in the database. I also wanted to be able to extend my sinkhole to collect additional analysis, automatically unpack malware in attachments, log out to elastic search etc etc. Because of these reasons I wrote [Sarlacc](https://github.com/scrapbird/sarlacc).

## Overview

Sarlacc is written in python3.5 using `asyncio`'s async/await model and has a basic plugin system that allows me to extend it to do all of the above features and more. Python was chosen as a language because of the massive number of RE libraries available for performing additional analysis / unpacking etc. It uses postgres as it's relational database and stores all attachments as documents in mongodb. This gives me the option to tag certain samples in the mongodb database and store additional metadata alongside.

## Plugin System

Sarlacc will load any python files dropped into `smtpd/src/plugins` as a plugin. There is an example plugin included with the repo in `smtpd/src/plugins/example.py`. To create your own plugins simply extend `SarlaccPlugin`, which is defined in `smtpd/src/plugins/plugin.py` and override any events which you would like to know about. Sarlacc is written in, for a full list of events you can override, take a look at the definition for `SarlaccPlugin`. Most events are asyncio coroutines and called with `await`, with only the `run` and `stop` events being called synchronously. `run` is called before the smtp server starts and should be used to start any long running jobs / threads that the plugin will require. `stop` is called after the smtp server has shut down and should be used to clean up any long running tasks started in `run`.

Keep in mind that Sarlacc uses async/await while writing your plugins and use `asyncio` supported libraries where possible, ensuring to yeild to the event loop with `await` where appropriate.

## Conclusion

Sarlacc is still a major work in progress and will be evolving over time as I add features that I want to it. There is very little in the way of documentation but I will be working on this as I get time. Feel free to contact me with any questions. I'm publishing this in the hopes that it is useful for someone who wishes to build something similar in their own labs.