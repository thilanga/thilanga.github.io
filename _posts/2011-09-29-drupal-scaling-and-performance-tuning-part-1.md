---
layout: post
title: Drupal scaling and performance tuning - Part 1
date: 2011-09-29
comments: true
tags: apache drupal memcached mysql nginx php pressflow
---

As WSO2Con 2011 was a huge hit in IT sector, I had to face a problem with [wso2.org (Oxygentank developer Portal)](http://wso2.org/).
Which site couldn't handle a large traffic constantly.

The old system we had was NGINX (Load Balancer) fronted 4 Drupal nodes which was running with Apache2 with master - master MySQL replication.

But during a high-traffic Our servers got more than 1 minute (average) time to respond to a request.
As NGINX's fail timeout for 45 seconds, users got 504 Gateway timeout most of the time. Same time Apache server load went above 30 - 70.
First as a quick solution we spinned new 2 servers and routed the load across them. That helped us to cater for few hours but
again there was a huge delay when site was loading. Then we found that MySQL server load also gone up and process-list has grown and
MySQL Server take time process queries than it was before. Root cause for this new problem was 6 drupal nodes had started stress
MySQL servers continually.

Then I started to ask some questioned by my self.

>OK, Then why we didn't see this new problem before we plug new nodes?
Answer was simple, Drupal nodes couldn't stress the DB as it got self killed (Frozen) during high traffics.

Before jump in the Apache, Our WSO2 Infrastructure team revisit our monitoring systems (Ganglia) and found that servers
were running on swap most of the time

As a solution I stated to tune up the Apache to avoid memory swapping problem. Then I saw that there was some miss configurations
which take more memories when Apache process start to handle the traffic. Due to that configuration server can easily excused.

`MaxClients` as was one of a major key parameter I had to change during the Apache memory optimization process.

####Apache Memory Optimization

+ Find the non-swapped physical memory Apache has used (**RES**)
    + Run top command and press shift + m to find highest  RES value which is Apache process use
+ Find the AVAILABLE APACHE MEMORY POOL.
    + Run `service apache2 stop` and type `free -m`
    + Note the used memory and Subtract it from **total** (This will give you the FREE MEMORY POOL)
    + Multiply MEMORY POOL by **0.8** to find the average AVAILABLE APACHE POOL (This will allow server  20% memory reserve for burst periods)
+ Calculate MaxClients
    + Divide AVAILABLE APACHE POOL by the highest **RES** memory used by Apache (Step 1). Now we have `MaxClients` number
    + Open `apache2.conf` and change `MaxClients` value
+ Other tweaks
    + Set `Keepalive On` (keep it **off** if you haven't keep 20% memory reserve. )
    + set `keepalivetimeout` to the lowest value (This will prevent connection hanging, If you experience high latency to your server, set it to 2-5 seconds)
    + Set your **Timeout** to a reasonable value like 10 - 40 (Keep it low)
    + set `MaxKeepAliveRequests` with in 70-150 (If you have a good idea about objects in a 1 page set it to match the object count)
    + Set `MinSpareServers` equal to 10-25% of `MaxClients`
    + Set `MaxSpareServers` equal to 25-50% of `MaxClients`
    + Set `StartServers` equal to `MaxSpareServers`
    + Set `MaxRequestsPerChild` somewhere between 400 - 700 (if you see rapid Apache child process memory use growth) to 10000
+ Apply Changes to Apache
    + Run `service apache2 start`

Once I restart our servers with above setting, server could handle 150% traffic from Apache end and server load was always below 2 (in a burst server-load was 1-2)

This solved my Apache hanging problem. Then I ran a another load test to verify my setting. I was amazed with the improvement but that haven't solved my problem.

server is taking bit long to response but I didn't see a connection drop.

>Now, What is wrong with my settings, Why server response is slow ? **MySQL!!**

I will discuss in part 2, how I addressed this problem .