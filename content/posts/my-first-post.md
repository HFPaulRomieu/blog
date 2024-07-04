+++
title = 'My First Blogpost: setting up my own blog'
date = 2024-06-29T12:34:55+01:00
draft = false
+++
## Introduction

I finally gave in to the FOMO of having my own blog. Things I like in blogs:


No one can comment.

Having my own blog was something that I had in my TODO list for quite a while. Deciding on the content though is of course a task of its own.
And because I am lazy, this first blog post is going to be about how I set up this blog!

![see what I did there?](/images/spiderman.jpeg)


#### Step 0
First things first: I was wondering whether I should be hosting this in a cloud provider's object storage service (like AWS S3) but then
I remembered something: I don't really like paying. Also, some cloud horror stories I come across sometimes (like [this one](https://www.reddit.com/r/webdev/comments/1b14bty/netlify_just_sent_me_a_104k_bill_for_a_simple/))
make me extra cautious. Plus on the bright side, I would understand how things work better without having the comfortable cloud abstractions in place.

#### Step 1
So I bought a raspberry pi that I could use as a server. Which is also something I wanted to buy either way (or I was simply looking for an excuse to buy one).

#### Step 2
Find a domain: johnmantios.fyi was a cheap option (not for long though...)

#### Step 3
Use a dynamic DNS service: because I am not paying my ISP for a static IP, I had to find a way to overcome this. I chose
a free service called no-ip which essentially points my router's dynamic IP address to a static hostname: no-ip checks
every 5 minutes for changes to my router's IP address. If the latter changes, then no-ip updates the hostname I have chosen
with the new IP address.

#### Step 4
Create a CNAME record: In the domain registrar from which I bought my domain, I added a CNAME record which forwards traffic
from my domain to my no-ip domain.

#### Step 5
Assign a static IP to my raspberry pi: In order to do that, I had to connect my raspberry pi to my router through an
ethernet cable. Every ethernet device has a unique MAC address. So I opened up my router's settings and I assigned my rpi's MAC
address a specific IP. Also known as a DHCP rule.

#### Step 6
Still in the router: add port forwarding rules. Redirect packets to specific ports from which your rpi will serve your html content.

#### Step 7
Sneaky step: make sure with your ISP that there is no carrier grade NAT sitting behind your router's IP. 

#### Step 8 
rpi: ssh into your rpi server and install nginx. Also execute certbot or acme commands in order to obtain your SSL/TLS certificate.
Of course, you can automate this in 1000 different ways but this greatly depends on the time and effort you want to put.

#### Step 9
serve your application: start nginx and serve your html files. In my case I am using hugo as a static site generator