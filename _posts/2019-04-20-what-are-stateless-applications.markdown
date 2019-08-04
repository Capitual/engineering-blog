---
layout: post
title:  "What are stateless applications?"
date:   2019-04-20 13:38:53 -0300
categories: jekyll update
comments: true
author: jesobreira
---

![jordan-harrison-1208586-unsplash](/img/posts/jordan-harrison-1208586-unsplash.jpg)

One of the most important aspects when developing an application that is going to be accessed by multiple users at once is to keep yourself online. The standard LAMP (Linux + Apache + PHP + MySQL) Stack that we've learned to easily install and configure may not be enough for the traffic we're expecting. It's ok, you've learned that, for such cases, and when vertical scaling (upgrading our server's resources) is not an option anymore, we create *replicas* of our server (horizontal scaling). Having multiple servers being able to perform the same actions that may be requested by the users, we have now just to split users among these replicas, and this is done by placing a load-balancer, which is a server (often a Nginx server set up as a reverse proxy) that does not perform any action (is completely static), but only guide users to different servers, keeping none of the servers under high load.

This sounds beautiful, but it also brings a few more concerns that, despite being quite simple, may take your whole service down if you do not pay attention on it. To put in just a few words: you have to master how load-balancer works before making use of one.

A load balancer is normally a reverse proxy. It's the only server hit by your users on your backend's structure. On each HTTP connection it receives, it forwards every request to one of the backend servers' replicas, based on a defined measure to avoid high-loading one of the servers (although, in several systems, the destination server is simply randomly selected). But all load-balancing usages will face one important requirement: the application must be *stateless*.

## What?

The Mr. Obvious Programming Dictionary defines *stateles* as an application that does not keep an *state*. Since it does not help a lot, let's first define *state*.

State is the current application's current procedure and already inserted data. Going a little deeper, it's the defined variables and their values, memory contents and current point of execution.

Obviously this definition changes depending on the context, as writing a stateless application does not mean you can't use variables or call functions. Stateless applications, in this context, are applications that does not store one user's state internally, and is able to perform or resume any action as soon as the user requests it, regardless of the action being initiated on the same server or in another replica.

For a better understanding, let's pick a good example of a stateful device: a calculator. Say you want to calculate 5+5 on it. You start by pressing 5, and the calculator's current state stores the current number. Then you proceed by pressing the plus sign, and the calculator's state stores that you want to perform a sum operation.

If you want to finish your calculation in another calculator, you're unable to, as the other calculator's state will not know that you've already pressed the digit "5" and the plus sign. You'll have to start it all over.

Also, if I want to take your calculator and do another calculation, it will not work either, as the calculator's state is storing your calculation's details. To perform my calculation, I'd have to wait for you to finish, or cancel your calculation.

A stateless application would not have these two issues, as the state would be *shared* among the two calculators, and when you picked the second calculator, it would already know the buttons you've previously pressed.

Now, let's do some coding to better understand it.

The application below stores the user's login procedure in a variable. It's stateful and can only serve one user at once, otherwise it will cause problems. Note that all user's trials are stored on the same variable.

```
let trials = 0
app.post('/login', (req, res) => {
	if (trials <= 3 && auth(req.body.username, req.body.password)) {
		res.send("welcome!")
	} else {
		trials++
		if (trials > 3) {
			res.send("Brute force blocked")
		} else {
			res.send("Wrong credentials")
		}
	}
})
```

The application below is also stateful, and does not share the state to its replicas. Although it would work well when running alone, it would be troubleful if running replicated, as the brute force control (`trials`) is done internally, on a per-replica basis. In other words, once an user tries logging in again, if the load balancer sends this user to a different replica, that replica would not have stored that user's trials.

```
let trials = {}
app.post('/login', (req, res) => {
	if (trials <= 3 && auth(req.body.username, req.body.password)) {
		res.send("welcome!")
	} else {
		if (trials[req.body.username]) {
			trials[req.body.username++
			if (trials[req.body.username] > 3) {
				res.send("Brute force blocked")
			} else {
				res.send("Wrong credentials")
			}
		}
		else {
			trials[req.body.username] = 1
			res.send("Wrong credentials")
		}
	}
})
```

**Note that the codes above aren't production-ready. They've been wrote for example purposes only and shall not be copied to production usage.**

## Why?

Creating a stateless application is needed when you're replicating your application behind a load balancer for a single reason: you can't tell whether the user's next HTTP request will be routed to the same replica. The load balancer could simply move that user away to another replica.

However, stateless applications may not always be applications that simply does not store any state, as this is not always possible. When developing a stateless application, you may have to keep states, but you have to ensure the state is shared among all your replicas. In this context, a stateless application is one that does not need information that cannot be obtained by the current replica which the user is connected to, in order to perform the requested action.

The problem starts when we find out that an application may be stateless in some environments, but non-stateless in others. For example, PHP allows us to use "sessions". Sessions are global variables that are kept across different requests, making it possible, for example, to store the logged-in user data. This is done, by default, by setting a cookie (which default name is "PHPSESSID") on the user's side, containing an unique ID, which the interpreter may use to gather all the stored information (session) for that user. This looks quite stateless, as the session information is not stored in any variable on the PHP source code.

```
<?php
session_start();
$_SESSION['test'] = 'hello world';

// now we can use $_SESSION['test'] even after another request

```

But, by default, PHP stores session information on files at the server's temp folder. If you replicate your service behind a load-balancer, it's highly possible that an user signs in in a server, and then gets routed to another server, which does not have the user's session information (and, therefore, thinks the user is logged off).

Luckly, PHP offers methods for us to customise how to store session information. The function [session_set_save_handler](https://php.net/session_set_save_handler) allows us to replace the default store-in-temp-folder behavior of setting and reading a PHP session to anything we want. In this case, we can use it to write and read sessions on a centralized storage engine, accessible by all the replicas, such as a database.

NodeJS applications might also suffer such issue. Often, when having to keep temporary user data (such as tokens, recovery codes or even some caching), we end up looking for ways to store such data locally, in flat files, on the server's file system. While this may work well on our local development environment (as this is hardly replicated), things start having troubles when moving to environments where the servers are replicated, such as the staging environment or (hopefully not) the production environment. Since one server's file system will not be available on all the replicas, other servers will not have that data when the load balancer routes the client against them. As a result, your application will look unresponsible.

![But it works on my machine](/img/posts/breaks-in-production-saythe-line-bart-but-it-works-on-40738442.png)

## How?

It's simple: when working with replicas, never store, in one machine, anything that other machines may need to perform well. Just like the PHP example mentioned above, we shall use a centralized storage engine, acessible by every machine on our structure, and make sure all these information is stored on it.

We could suggest some relational database, like MySQL or PostgreSQL, since such data is often relational (as this is client-related). But if you have a high-traffic application, you may already know that it's better to avoid hitting the RDBMS with temporary data, as this is often a more expensive storage. Also, due to the relational nature, several comparisons are required for filtering results, which gives us a higher response time when requesting such data.

For this use case, key-value storages like Memcached and Redis are better suitable. As they have less tasks to perform for retrieving one value (since there aren't tables or columns, and everything is a "key" with a "value"), their use is highly recommended when dealing with temporary data that is often accessed. Also, these engines often offer a way to control each key's expiration time makes it less likely for leaks and accidental persistence of unnecessary data.

Obviously, since these storages are unrelational and only key-value based, you cannot fetch, at once, the data related to an specified user. And obviously, you cannot store a variable named simply "login_attempts", as it will be valid for all your clients. However, you can *namespace* it (for example: `login_attempts.[user id]` or `login_attempts.[IP address]`) and the problem is gone.

Let's see an example?

Let's use [cacheman-redis](https://www.npmjs.com/package/cacheman-redis) to convert the code we've been dealing with above to a multireplica-safe one:

```
let CachemanRedis = require('cacheman-redis')
let cache = new CachemanRedis(/* redis connection details */)
app.post('/login', (req, res) => {
 cache.get('loginattempts:'+req.body.username, (err, trials) => {
  if (trials <= 3 && auth(req.body.username, req.body.password)) {
   res.send("welcome!")
  } else {
   if (trials) {
    cache.set('loginattempts:'+req.body.username, trials++)
    if (trials > 3) {
     res.send("Brute force blocked")
    } else {
     res.send("Wrong credentials")
    }
   }
   else {
    cache.set('loginattempts:'+req.body.username, 1)
    res.send("Wrong credentials")
   }
  }
 })
})
```

While this code may be improved in several ways (you could promisify the caching functions; also, blocking the user for login attempts may be a little awkward in most of the situations), now our code is able to recover, from a centralized Redis database, the previous login attempts of the user currently trying to login.
