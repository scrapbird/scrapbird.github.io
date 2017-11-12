---
layout: post
title: "Chasing Necurs: The Beginning"
---

A few months ago I was trying to decide what my next big side project would be and I decided to set my sights on Necurs. Necurs, for those unfamiliar is a botnet being used largely as a carrier for Locky and Dridex by way of spam campaigns. This was an attractive target for me because of the future potential to generate some really useful threat intel and get brand new samples of Locky and Dridex while they're still hot.

To capture spam from an infected host the general idea was to create a virtual network consisting of the following hosts:

- Pfsense router
    - Redirect any outgoing SMTP traffic to the SMTP sinkhole
    - Block access to any hosts not needed by the malware
- SMTP sinkhole
    - Running an email sinkhole (currently running [mailslurper](https://github.com/mailslurper/mailslurper/) while I implement my custom SMTP sinkhole)
- Monitoring workstation (for packet capturing etc)
- Windows 7 infected necurs host

## The Network

First I set up an ESXi host with the 4 VMs detailed above and arranged them into a few networks. The infected necurs host and the monitoring workstation are inside the "redzone". I call this network the "redzone" so I don't accidentally connect it to my real network. The SMTP sinkhole resides in my safe "greenzone" network. The topology of the network looks like the following (because who doesn't like a shitty network diagram):

![redzone greenzone network diagram](/images/necurs-network-diagram.png "redzone greenzone network diagram")

The router is set up with three network adapters, one connected to the redzone, one connected to the greenzone and the other to the internet (well, it eventually gets there). The SMTP sinkhole has a single adapter connected to the greenzone and the other two hosts have a single adapter which is connected to the red zone. The pfsense router is in charge of handing out IP addresses on both networks. This way all traffic must pass through the router before it gets anywhere else, and I easily have full control of packet flow.

Additionally, the pfsense router is configured with a NAT rule which will forward all SMTP traffic to my SMTP sinkhole server. This is the only way traffic can pass from greenzone to redzone.

## SMTP Sinkhole

The SMTP sinkhole server is running [mailslurper](https://github.com/mailslurper/mailslurper/), which is configured to simply store all mail it receives in a database and not forward it anywhere. This is only temporary while I complete my more advanced SMTP sinkhole server, which will automatically classify and store email attachments and generate some pretty cool intel, which I will make available at [intel.devit.co](http://intel.devit.co/).

## Monitoring Workstation

The monitoring workstation is pretty simple, just an ubuntu machine running wireshark. The only thing that might need to be mentioned here is that the redzone network is configured to allow promiscuous mode so that packet capture is possible.

## Infected Necurs Host

The infected necurs host is where it got complicated. Necurs contains a few anti VM / anti debugger techniques that made it harder to debug and analyze. Because of my future intel generating plans I didn't want to simply infect a host and let it run, I wanted to be able to instrument the binary to automatically track new modules loaded, configs, email templates etc.

Necurs also contains a rootkit, with a driver and a usermode module that communicate with each other. This makes it harder to debug as well. As such I spent some time reversing the sample to patch out the rootkit, anti VM, and anti debugger code. It isn't perfect and the sample still does create some junk files on the OS but it can be run as a standalone executable with a debugger attached, enabling some pretty cool instrumentation in the future.

Here is the necurs sample running as a standalone executable:

![necurs standalone execution](/images/necurs-standalone.png "necurs standalone execution")

## Patched Sample

I have made this patched sample available to the community for download on Hybrid Analysis and it can be found here: [https://www.hybrid-analysis.com/sample/d288408c2d7f2ad17a92df8a60384ff608224aabd6c43443b7ca3ff3fd61a103?environmentId=100](https://www.hybrid-analysis.com/sample/d288408c2d7f2ad17a92df8a60384ff608224aabd6c43443b7ca3ff3fd61a103?environmentId=100)

## The Outcome

There is still a lot to do but I am very happy with the outcome of my work so far. I am currently capturing SMTP traffic as I type this and the sample is running smoothly. My monitor shows all C2 communication and SMTP traffic flowing from the box, as can be seen here:

![necurs packet capture](/images/necurs-pcap.png "necurs packet capture")

And mailslurper is storing all these emails for me (although not correctly decoding the base64 email body):

![necurs email log](/images/necurs-emails.png "necurs email log")

The current spam campaign seems to be simply fishing for active email accounts, all emails contain a short message attempting to entice the recipient into replying to them and leaves an email address for them to reply to. They all look similar to the following:

>Hi, honey!
>The main thing for us is to trust and to hope. When I close my eyes, I see you.
>I send you a lot of kisses. Write me please! My email elenabirgit918@rambler.ru

## Future Goals

I have a lot of goals I still would like to achieve, but the main goals are as follows:

- Automatic detection and reporting of new necurs modules
- Automatic categorization of email attachments
- Automatic unpacking of Locky / Dridex samples and config extraction
- Collecting network / file IOCs including DGA domains and the IP addresses the bot will connect to

I will be posting updates to this blog as I progress, this has been an ongoing project for me and I will continue to work on it as time allows (this isn't my day job).
