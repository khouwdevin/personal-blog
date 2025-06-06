---
title: 'Gitomatically'
description: 'Gitomatically is an app designed to implement CI/CD with ease.'
publishedAt: '2025-06-06'
updatedAt: ''
---

After tried to learn and use Linux Server ([A New Playground's blog](https://khouwdevin.com/blog/anewplayground)), I really enjoy to have a server where people starting to use my Discord Bot too. But, one issue that really tiring I need to log in to the server and redeploy manually. After searching a quite long time, I found that Github use webhook to tell us the repository events, however it might be hard and it will require a lot of works since each apps have different configuration and deployment's steps. Then, I came up with this [Gitomatically](https://github.com/khouwdevin/gitomatically) app idea and coincidentally it has been a long time I want to learn Go and finally I found the right moment to learn it.

# Purpose

The requirements are very simple, I want the app to be able pull from Github for now and it can do simple command sequences. It is quite simple, right? How about url, git clone url, which branch and many more? I think it is suitable to use yaml as many apps use it.

The problems are solved, what about the app scope? For now, it can be used by one server. The main thing is it is intended for hobby, so, generally people only have one server. But, do not worry, things might be changed because I use this app actively.

# How it works

Let's talk about how this app works and excuse my code if it is bad, it is my first time write Go.

## Framework

A lot of people say that Go does not need framework, the built in library can do simple thing, but for me, using a framework will help me to progress faster which I do not need to think about the unpredictably problems, I just need to focus on the logic, so, I use [Gin-Gonic](https://gin-gonic.com/).

## Structure

In **Gitomatically**, I only use two API routes, the first one is `/`, it will return greetings and the second one is `/webhook`, it will return ok to Github server and will process from the configuration.

```go
// main.go

router := gin.Default()

router.GET("/", func(c *gin.Context) {
  c.JSON(http.StatusOK, gin.H{"message": "Welcome to gitomatically!"})
})

router.POST("/webhook", middleware.GithubAuthorization(), controller.WebhookController)

router.Run(fmt.Sprintf(":%v", env.Env.PORT))
```

## Configuration

**Gitomatically** use yaml as configuration file, it consists of url, clone url, branch, path and commands, each of keys are mandatory to be filled. Let's take a look on `conf.yaml` example.

```yaml
# conf.yaml

repositories:
  example.com:
    url: https://github.com/example/example.com
    clone: git@github.com:example/example.com.git
    branch: main
    path: /home/khouwdevin/apps/example.com
    commands:
      - docker compose up --build -d
```

App name can be named as you like and commands do not need to be filled. Commands can be used for build sequences or other things after do git pull.

Additionally, you need to add `.env` and ssh config in `.ssh/config`, so the app can work properly. For complete guide to use the app you can see on README file in [Gitomatically repo](https://github.com/khouwdevin/gitomatically).

## Core

**Gitomatically** is designed to be effortless in maintaining apps, if you do not need to apply some configurations, you can write `conf.yaml` and **Gitomatically** will handle it for you. If you need to set it up first, you can do it then **Gitomatically** will help you to pull and redeploy.

# Problems

I have been a Windows user since forever which make me not have experience in Unix. At first, I want to use `docker` because it is simple and works perfectly with just one command, then I realized I cannot use `docker` because the purpose of `docker` itself. So, I do not have any choice but to use systemd, it was a long time to figure out simple thing, I searched through many threads but did not find a single solution to make git ssh work. But I did not stop there, at the end I was able to solve it with add ssh config.

Second problem is about `path`, I though it will be simple, but it is not, `path` has strict rules as far I know in Go especially. Path cannot contain addition of `/` or `\` at the end of last directory so it should be like `/home/gitomatically/apps/gitomatically` or `C:\Desktop\gitomatically`, as far I remember in node, it is not like this, I might be wrong tho.

Last, `exec` in Go is different, you need to pass the first argument separately from the rest of the command, because Go search the app through the first argument so if we pass `git pull` to the first argument, Go will search `git pull` app which it will not be exist, we need to pass `git` and the rest can follow.

Although a lot of problems happened and it is not related to the core app, I learned a lot, hopefully, I solved the problem completely.

# Conclusion

I am happy with the app, it is fulfill my needs for now. I will keep developing **Gitomatically**, where there are some features I will need in the future.

If you find this app interesting, you can visit the repo [here](https://github.com/khouwdevin/gitomatically), also if you guys have feedbacks you can leave a comment or open an issue on the repo.