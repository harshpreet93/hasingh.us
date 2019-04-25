+++ 
date = "2019-04-24"
title = "Runtime Code Loading in Golang"
slug = "runtime-code-loading-golang" 
tags = []
categories = []
series = ["golang"]
+++

## The motivation behind runtime code loading
In a previous job, the team that I was on mostly ran quite a large monolithic application, that was depended on by many other teams. We deployed the application quite frequently for such a large monolith, usually once or twice a day after or just before our main business hours. At times, we needed to be able to iterate on this application faster, during business hours, and without doing a full redeploy of the application. This being a large monolith, a deploy meant a few minutes of downtime, which, while not forbidden, was not encouraged either, since multiple teams and people relied on the application to get their work done during business hours. In order to be able to run code as part of the monolith without having to redeploy it, we relied on runtime code loading. Here's what the workflow looked like, a dev would write some code, get it merged into the repo. At the same time, there would be a background thread in the monolith that would keep pulling the repo at regular intervals of a few seconds or so and keep it up to date. Usually, at this step, most people would think the next steps are to build the code into an artifact and deploy it to a machine, but in our case, pulling the repo constantly was the build and deploy step all combined into one. Once the monolith had the code, it became available to run and the monolith could command an embedded interpreter to run the code. A dev could run it manually by pressing a button in a web ui or he/she could just put a cron expression at the top and the monolith would then schedule it according to that. This is not a new idea. Many battle tested and insustry standard applications rely on a similar methodology. HAProxy can be scripted with the lua programming language and can be compiled with an embedded lua interpreter (https://github.com/haproxy/haproxy/blob/master/doc/lua.txt). Nginx has a similar story (https://www.nginx.com/resources/wiki/modules/lua/). The TL;DR of this is that this is not a fringe idea and it can do wonders for productivity in environments where redeploying entire application/s every few minutes is not a viable option.
## How do we do this in Golang?
Since the monolith I mentioned above was Java based, I started wondering how a similar architecture could be enabled with Golang. Basically, I want to be able to commit my Golang "scripts" a repo and then be auto discovered, loaded into the supervisor app and be runnable, all without going through the build and deploy step. Interestingly, Go 1.8, which is a fairly recent development, got a feature (https://golang.org/pkg/plugin/) to make exactly this easier! So let's get down to it and create a POC application that brings the embedded scripting-like experience to the world of Go.


