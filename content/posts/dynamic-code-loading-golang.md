+++ 
date = "2019-04-24"
title = "Runtime Code Loading in Golang"
slug = "runtime-code-loading-golang" 
tags = []
categories = []
series = ["golang"]
+++

## The motivation behind runtime code loading
In a previous job, the team that I was on mostly ran quite a large monolithic application, that was depended on by many other teams. We deployed the application quite frequently for such a large monolith, usually once or twice a day after or just before our main business hours. At times, we needed to be able to iterate on this application faster, during business hours, and without doing a full redeploy of the application. This being a large monolith, a deploy meant a few minutes of downtime, which, while not forbidden, was not encouraged either, since multiple teams and people relied on the application to get their work done during business hours. In order to be able to run code as part of the monolith without having to redeploy it, we relied on runtime code loading. 

