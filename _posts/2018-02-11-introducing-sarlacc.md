---
layout: post
title: "Introducing: Sarlacc - An SMTP Sinkhole"
---

![sarlacc database](/images/sarlacc-database.png "sarlacc database")

A few months ago I published a [blog post](/chasing-necurs/) on my necurs tracker and in it I described my spam collection setup in my lab. My setup is still very much the same except for one thing: the SMTP server. Previously I was using [mailslurper](https://github.com/mailslurper/mailslurper) but found that it was inadequate for my purposes. This was mainly because of the way the database was structured, every mail item that mailslurper collects gets stored with no relational structure so you end up with millions of copies of the same data being stored in the database. I also wanted to be able to extend my sinkhole to collect additional analysis, automatically unpack malware in attachments, log out to elastic search etc etc. Because of these reasons I wrote [Sarlacc](https://github.com/scrapbird/sarlacc).

## Overview

Sarlacc is written in python3.5 using `asyncio` and has a basic plugin system that allows me to extend it to do all of the above features and more. Python was chosen as a language because of the massive number of RE libraries available for performing additional analysis / unpacking etc. It uses postgres as it's relational database and stores all attachments as documents in mongodb. This gives me the option to tag certain samples in the mongodb database and store additional metadata alongside.

It is easy to get up and running with Sarlacc on any system with docker and docker-compose installed, simply run:
```
docker-compose up
```

See the [`README.md`](https://github.com/scrapbird/sarlacc/blob/master/README.md) for more details.

## Plugin System

Sarlacc will load any python files dropped into the [`smtpd/src/plugins`](https://github.com/scrapbird/sarlacc/tree/master/smtpd/src/plugins) directory as a plugin. There is an example plugin included with the repo in [`smtpd/src/plugins/example.py`](https://github.com/scrapbird/sarlacc/blob/master/smtpd/src/plugins/example.py). To create your own plugins simply extend `SarlaccPlugin`, which is defined in [`smtpd/src/plugins/plugin.py`](https://github.com/scrapbird/sarlacc/blob/master/smtpd/src/plugins/plugin.py) and override any events which you would like to know about. For a full list of events you can override, take a look at the definition for `SarlaccPlugin`. Most events are asyncio coroutines and called with `await`, with only the `run` and `stop` events being called synchronously. `run` is called before the SMTP server starts and should be used to start any long running jobs / threads that the plugin will require. `stop` is called after the SMTP server has shut down and should be used to clean up any long running tasks started in `run`.

Keep in mind that Sarlacc uses async/await while writing your plugins and use `asyncio` supported libraries where possible, ensuring to yield to the event loop with `await` where appropriate.

## Todo

There is still a lot more I want to do with this tool. One of the main things still needed is to expose an internal API layer to the plugins so that they can access and modify the data stored in postgres and mongodb. This would add the ability to tag certain attachments with metadata gained during analysis. Then plugins could scan attachments with yara and record the matched rules alongside the sample, for example.

I am also planning on including a web application to interface with the data, but for now you have to interface with the databases directly.

## Conclusion

Sarlacc is still a major work in progress and will be evolving over time as I add features that I want to it. There is very little in the way of documentation but I will be working on this as I get time. Feel free to contact me with any questions. I'm publishing this in the hopes that it is useful for someone who wishes to build something similar in their own labs.