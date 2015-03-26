---
layout: post
title: My Perfect Mobile IRC Client
---

If you're a software developer there is a good chance you spend a fair amount of time on an irc server. For me irc has been there since I first tried my hand at a few little batch scripts on an old windows machine. I spent hours on various servers and writing cool little bots is what got me into real programming. I still remember the feeling I got when I wrote my first little script to connect to another machine over the network and seeing that message pop up on the screen from the computer in the next room. I was hooked. But this isn't a post about how I fell in love with programming, it is a post about my perfect irc setup on my android phone.

When I got my first smart phone the first thing I wanted to do was connect to my network and hop on my usual channels to start chatting. So I checked the play store to see what the irc apps looked like. I tried a few and eventually ended up using AndChat. And to be fair it is an okay client but it just never felt right for me. I'm more of an irssi guy. Recently I finally got around to remedying the problem, I just ssh into my server from my phone and drop into a shell running irssi.

I know this seems like a stupid solution to some and many of you will stop reading this post right now. That's okay. But if you've never quite been happy with your mobile irc client for similar reasons just hear me out.

When using this method we get a couple of benefits for free:

* Strong encryption through ssh
* Never miss a message using ZNC
* Your irc admin will not know your IP address
* Uses your already existing setup

Now I'm sure there are ways of compiling irssi for ARM and setting it up correctly but I already had my trusty VPS running ZNC and my preexisting irssi config so I thought, "why not use that?"

The first piece of the puzzle is finding an ssh client that suits you, it doesn't have to be full featured and awesome because all you will ever be doing in it is typing messages and pressing enter. I use [JuiceSSH](https://play.google.com/store/apps/details?id=com.sonelli.juicessh&hl=en). JuiceSSH is a pretty good ssh client and the best part is it supports switching channels in irssi via swiping across the screen (and is turned on by default).

Now that you have that installed, lets set up the VPS. This is not a post about how to set up irssi or ZNC, there are already a huge number of tutorials out there for that and I'd rather not type out what has already been said multiple times. Log into your server and lets create an account for you to use as your irc login:

{% highlight bash %}
# create the user
sudo useradd ircuser
# give them a password
sudo passwd ircuser
{% endhighlight %}

Create a home directory for this user and copy our irssi config files into it:

{% highlight bash %}
# create the users home directory
sudo mkdir /home/ircuser
# give the new user access to the directory
sudo chown ircuser:users /home/ircuser

# copy our irssi config files over
sudo cp -R /home/YOURACCOUNT/.irssi /home/ircuser/.irssi
# give ircuser access to the files
sudo chown -R ircuser:users /home/ircuser/.irssi
{% endhighlight %}

When we ssh into our server using this account we don't want to have to run irssi ourselves, so lets change the users shell to be irssi:

{% highlight bash %}
sudo chsh -s `which irssi` ircuser
{% endhighlight %}

Now all that is left to do is set up your new ssh account on JuiceSSH and you have yourself your nice familiar irssi setup on your phone using strong encryption and all the customizable scripts you are used to. Perfect.
