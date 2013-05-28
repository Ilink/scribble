---
layout: post
title: Scrape Everything
date: 2013-05-27 17:03:00
disqus: n
---

League of legends is a very popular game. I happen to be one of the many millions of players. Recently, I have been getting interested in machine learning. I decided to combine both interests. That's where I got the idea to create a champion suggestion engine. 

For the uninitiated, League of Legends is a 10 player game, with 2 teams of 5 players each. Everyone picks their character (champion) within the confines of this team setting. Different team compositions have wildly different rates of success. It's pretty easy to create a team that will have zero chance of winning. The purpose of my recommendation engine is to fill in the gaps of a team composition. 
For instance, let's say a team contains the follow champions:

 1. Kog'maw 
 2. Taric
 3. Nocturne
 4. Zac

Who would be the best champion to fill the last spot? 

Data harvesting was the first step. League of legends has no public API. Some 3rd parties also have big databases of League data. Sadly, those don't really have APIs either. Easy solutions are no fun anyways.

Scrape Everything
-----------------

After deciding on a particular third party League website, I went to work. Their pages looked pretty easy to parse. Very consistent markup, everything in it's place. I wasn't sure what tools to use, so I began with PhantomJS. That was a mistake. PhantomJS was tricky and I was impatient. CasperJS was an option, but I just didn't feel like it. 

Beautiful Soup was the next option. I really love Python and wanted a chance to use more of it. Beautiful Soup wound up being extremely easy to use, even on my first try. A few hours later, I had a rough scraper. A few more hours and I had most of what I needed. I could take my parser and get the last 10 matches for any given player. 

Destroy my Bandwidth
--------------------

Parser ready? Let's get some data! Let's also hit the max number of file descriptors! Urllib2 was probably not the best choice. I really should have just used [Urllib3][1] / [Requests][2]. Both of those have nice HTTP pooling. However, I am a masochist, so I made my own solution with gevent and urllib2. It was fun, sort of!

Essentially, the URL fetcher will limit simultaneous connections to a fixed number. To that end, it will only spawn a limited number of gevent "greenlets" at a time. The URLs themselves are stored in a RabbitMQ queue. In order to limit the overhead with connecting/reconnecting to the RabbitMQ server, the connection is severed until all the current jobs have been completed.

Enter RabbitMQ
--------------

Initially, grabbing all the web data was a bottleneck. At first, all my URLs were being fetched syncronously. This was improved with the code described in the previous section. However, I wanted to make a better pipeline. With an eye on future complexity, I decided to use a message queue server. I hadn't used [RabbitMQ](http://www.rabbitmq.com) before, so I decided to give it a try. 

RabbitMQ also gave me some resiliency for the web data. If a process crashed midway, the script could resume without having to re-fetch all the web data.

Pika provided the connection to the server. To simplify things further, I wrote a simple wrapper on top of Pika. This resulted in two classes: receiving and publishing. Pika made things pretty easy, but I had a few hiccups along the way. At one point, I was registering a debug logger with Pika. I wanted it to redirect all logging information to my main log files. I set the warning level to critical. Pika logs the body of all messages being sent to the server. This meant about 5000 web pages were written to a single log file. My VM ran out of space then RabbitMQ exploded.


More Architecture
-----------------

I wanted to create an easily-parallelizable architecture. Furthermore, I wanted to increase the modularity in my code. A slightly simplified version of the architecture looks like this:

![enter image description here][3]

Aside from a simple pipeline, this architecture also allows for some parallelism among multiple python processes. This was an easy win. The architecture is more modular and more performant. A simple bash script kicks off the individual processes. Each independent script utilizes the same bootstrapping. This provides them with the appropriate log files and access to the proper config file. It also allows the individual processes to accept the same CLI parameters. 

In the future, the Web Data Queue could perform more sophisticated routing. This would deliver data to the appropriate parser. However, there is currently only a single parser. RabbitMQ has all sorts of sophisticated routing that I haven't gotten to touch yet.

Next
----

Currently, I think I have enough match data to begin analysis. However, I am still new to statistical machine learning techniques. I am currently enrolled in the awesome [Coursera][4] ML class. The material from the class has already helped me get started on my analysis. I'll leave the details for the next post.


  [1]: https://github.com/shazow/urllib3
  [2]: http://docs.python-requests.org/en/latest/
  [3]: https://s3-us-west-2.amazonaws.com/blog-img/league-of-stats-arch.png
  [4]: https://www.coursera.org/