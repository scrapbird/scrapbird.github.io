---
layout: post
title: "Writing Sarlacc Plugins"
---

![sarlacc malshare](/images/sarlacc-malshare.png "sarlacc log output")

Since [releasing Sarlacc](/introducing-sarlacc/) it has received a lot more attention that I expected. It is still in it's infancy and even though I work on it as much as I can I haven't written any real documentation for it (besides python docstrings). Because of this I figured I'd write a little tutorial on writing a Sarlacc plugin.

## Overview

Sarlacc will load any python files found in the `smtpd/src/plugins` directory as a python module that extends the `SarlaccPlugin` class, defined here: [https://github.com/scrapbird/sarlacc/blob/master/smtpd/src/plugins/plugin.py](https://github.com/scrapbird/sarlacc/blob/master/smtpd/src/plugins/plugin.py).

You can also write your plugins as a python module in it's own directory, as we will do today. This means that all we need to do is drop our plugin into the correct directory and restart Sarlacc and it will be running.

We will be writing a plugin to upload any previously unseen email attachments to [Malshare](https://malshare.com/).

## Getting Started

Okay so first thing is first, let's create a directory for our plugin and create the `__init__.py` file:
{% highlight bash %}
cd smtpd/src/plugins
mkdir sarlacc-malshare
cd sarlacc-malshare
touch __init__.py
{% endhighlight %}

Next lets extend the `SarlaccPlugin` class:

{% highlight python %}
from plugins.plugin import SarlaccPlugin

class Plugin(SarlaccPlugin):
    def run(self):
        self.logger.info("Loading malshare plugin")

    async def new_attachment(self, _id, sha256, content, filename, tags):
        self.logger.info("Uploading new sample to malshare. SHA256: %s", sha256)
{% endhighlight %}

We are overloading the `new_attachment` method from the base `SarlaccPlugin` class here so that our plugin will get notified whenever a previously unseen attachment is detected. This method will usefully pass in any data you will need about the attachment, including the raw file data, filename etc. To see which other methods are available to notify our plugins of events see the [plugin.py](https://github.com/scrapbird/sarlacc/blob/master/smtpd/src/plugins/plugin.py) doc strings.

The Malshare API requires an API key, so we will create a config file and load the key from there. Create a file in your plugin directory named `malshare.cfg` with the following contents (replacing API_KEY with your API key):

{% highlight plaintext %}
[malshare]
key = API_KEY
{% endhighlight %}

Now we need to read this config in our plugin. Let's add some code to do this, modify your plugin code to look like the following:

{% highlight python %}
from plugins.plugin import SarlaccPlugin
from configparser import ConfigParser
import os

class Plugin(SarlaccPlugin):
    def run(self):
        self.logger.info("Loading malshare plugin")

        # Read config
        self.config = ConfigParser()
        self.config.readfp(open(os.path.join(os.path.dirname(os.path.abspath(__file__)), "malshare.cfg")))
        self.config.read(["smtpd.cfg",])


    async def new_attachment(self, _id, sha256, content, filename, tags):
        self.logger.info("Uploading new sample to malshare. SHA256: %s", sha256)
{% endhighlight %}

The last thing left to do is to actually upload the sample:

{% highlight python %}
from plugins.plugin import SarlaccPlugin
from configparser import ConfigParser
import os
import requests

class Plugin(SarlaccPlugin):
    def run(self):
        self.logger.info("Loading malshare plugin")

        # Read config
        self.config = ConfigParser()
        self.config.readfp(open(os.path.join(os.path.dirname(os.path.abspath(__file__)), "malshare.cfg")))
        self.config.read(["smtpd.cfg",])


    async def new_attachment(self, _id, sha256, content, filename, tags):
        self.logger.info("Uploading new sample to malshare. SHA256: %s", sha256)

        requests.post(url="https://malshare.com/api.php?api_key={}&action=upload".format(self.config["malshare"]["key"]),
                files={filename: content})
{% endhighlight %}

And there you have it, our plugin will upload any new attachments to Malshare.

Keep in mind this will pause the execution of Sarlacc while the upload is happening, so for large files this is not a good idea to do while under heavy load as for each new attachment execution will pause for a few seconds. To combat this we could use an HTTP client that makes proper use of `asyncio`, this is mostly to be used as an example.

Full code for this plugin is available on github: [https://github.com/scrapbird/sarlacc-malshare](https://github.com/scrapbird/sarlacc-malshare).