+++ 
date = "2019-04-24"
title = "Runtime Code Loading in Golang"
slug = "runtime-code-loading-golang" 
tags = []
categories = []
series = ["golang"]
+++

## The motivation behind runtime code loading
In a previous job, the team that I was on mostly ran/developed quite a large monolithic application, that was depended on by many other teams. We deployed the application quite frequently for such a large monolith, usually once or twice a day after or just before our main business hours. At times, we needed to be able to iterate on this application faster, with a REPL-like cadence in the production environment. This being a large monolith, going thorough a full merge, test, build, deploy loop, would mean that your changes wouldn't show up in production for atleast 20 minutes. In order to be able to run code as part of the monolith without having to redeploy it, we relied on runtime code loading. 

Here's what the workflow looked like, a dev would write some code, get it merged into the repo. At the same time, there would be a background thread in the monolith that would keep pulling the repo at regular intervals of a few seconds or so and keep it up to date. Usually, at this step, most people would think the next steps are to build the code into an artifact and deploy it to a machine, but in our case, pulling the repo constantly was the build and deploy step all combined into one. Once the monolith had the code, it became available to run and the monolith could command an embedded interpreter to run the code. A dev could run it manually by pressing a button in a web ui or he/she could just put a cron expression at the top and the monolith would then schedule it according to that. 

This is not a new idea. Many battle tested and industry standard applications rely on a similar methodology. HAProxy can be scripted with the lua programming language and can be compiled with an embedded lua interpreter (https://github.com/haproxy/haproxy/blob/master/doc/lua.txt). Nginx has a similar story (https://www.nginx.com/resources/wiki/modules/lua/). The TL;DR of this is that this is not a fringe idea and it can do wonders for productivity in environments where redeploying entire application/s every few minutes is not a viable option.
## How do we do this in Golang?
Since the monolith I mentioned above was Java based, I started wondering how a similar architecture could be enabled with Golang. Basically, I want to be able to commit my Golang "scripts" a repo and then be auto discovered, loaded into the supervisor app and be runnable, all without going through the build and deploy step. Interestingly, Go 1.8, which is a fairly recent development, got a feature (https://golang.org/pkg/plugin/) to make exactly this easier! So let's get down to it and create a POC application that brings the embedded scripting-like experience to the world of Go.

The first step is we need to keep an up to date copy of the repo that contains the plugin code. Luckily, there's an excellent library that does exactly that (https://github.com/src-d/go-git). Here's the bulk of the code that clones the plugin repo:
```
r, err := git.PlainClone(plugin_source_path, false, &git.CloneOptions{
		URL: repo,
})
```
	
Now, once we have a local copy of the code, we need to build those into `.so` plugin files. There's multiple slightly different ways of doing this. I ended up just putting a `go generate` directive at the top of each of my plugin source files and just running `go generate` programatically. Here's what a simple plugin file looks like:

```
//go:generate go build -buildmode=plugin

package main

import "fmt"

func Run() {
	fmt.Println("hello, world from pluginA")
}
```

Now, we can start actually compiling and loading the plugins:

```
func ReloadPlugins(pluginFolder string) []*plugin.Plugin {
	log.Println("loading plugins in plugin folder ", pluginFolder)
	srcFiles := findAllSourceFiles(pluginFolder)
	if loadedPluginHashes == nil {
		loadedPluginHashes = hashset.New()
	}
	for _, filepath := range srcFiles {
		log.Println("found ", filepath)
		builtPath, err := buildPlugin(filepath)
		if err == nil {
			//in order to not load the same file over and over again let's store the md5 hash of the plugin we just loaded
			hash, err := getMD5(filepath)

			if err == nil && loadedPluginHashes.Contains(hash) {
				log.Println("plugin at ", filepath, " with hash ", hash, " has already been loaded")
				continue
			}

			pluginLoaded, err := loadPlugin(builtPath)
			if err != nil {
				log.Println("unable to load plugin ", filepath)
				continue
			}
			plugins = append(plugins, pluginLoaded)
			loadedPluginHashes.Add(hash)
		}
		log.Println("number of loaded plugins ", len(plugins), " hash cache size ", loadedPluginHashes.Size())
	}
	return plugins
}
```
One of the problems that we must deal with is that, once a plugin has been loaded, if we keep reloading it, we're going to have a memory leak, the severeness of which would depend on how many/how large the plugins you're loading are. In order to mitigate this, we can just check the md5 hash of the plugin file we're loading and if it's already been loaded before, we can skip reloading it.

Once we have loaded the plugins, it's quite trivial to lookup a symbol and call it if it's a function:
```
for _, plugin := range loadedPlugins {
		v, err := plugin.Lookup("Run")
		if err != nil {
			log.Println("error finding Run symbol in plugin")
		}
		v.(func())()
}
```
Here's what a sample run of the finished product looks like:

```
2019/04/26 22:24:16 found  plugin_source/plugins/pluginD/pluginD.go
2019/04/26 22:24:16 wd is  /Users/harsh/code/go-dynamic-code-loading-blog
2019/04/26 22:24:17 file  plugin_source/plugins/pluginD/pluginD.go  hash is  a0e02dbb38136a789c410634b198eb71
2019/04/26 22:24:17 plugin at  plugin_source/plugins/pluginD/pluginD.go  with hash  a0e02dbb38136a789c410634b198eb71  has already been loaded
2019/04/26 22:24:17 found  plugin_source/plugins/pluginE/pluginE.go
2019/04/26 22:24:17 wd is  /Users/harsh/code/go-dynamic-code-loading-blog
2019/04/26 22:24:17 file  plugin_source/plugins/pluginE/pluginE.go  hash is  49d9de3ed224a3525bec3549d087bfc1
2019/04/26 22:24:17 plugin at  plugin_source/plugins/pluginE/pluginE.go  with hash  49d9de3ed224a3525bec3549d087bfc1  has already been loaded
hello, world from pluginA
hello, world from pluginB
hello, world from pluginC
hello, world from pluginD
hello, world from pluginE
```

All the code for this is available in the repo https://github.com/harshpreet93/go-dynamic-code-loading-blog. It can just be run by cloning and running `go run main.go` from within the repo root.

## Reflections


I've been pleasantly surprised by how easy it was to enable this embedded-scripting/plugin-based architecture in Go. Of course, all approaches have flaws/downsides, and one of the things I didn't mention above is that once a plugin has been loaded, it can't be unloaded. So, let's say we're using the above architecture in production and a developer commits a buggy plugin to production and it goes out and starts breaking things. If a fix is committed, the hash will change and the "new" plugin will be loaded, but the old one won't be unloaded. The mitigation for this would be to actually restart the app, which probably defeats the entire purpose/advantage of having this sort of architecture. The other approach, which is slightly more complex, but requires no downtime, would be to have "internal" garbage collection on the loaded plugins. Since we cannot actually unload the "old", buggy plugin, we could just "blacklist" it if we don't see it in source, and just not call it. This way, we can keep it loaded, but also have it be unreachable all without having any downtime. That particular enhancement is not in this POC, but I might revisit this in the next few days/weeks and implement that.