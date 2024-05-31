---
title: Opening Pull Request links from Terminal
author: Matthew
date: 2024-05-31T14:47:54+01:00
---

Recently I changed roles which meant leaving GitHub behind and using Bitbucket. There's many reason why the Git provider switch made me sad but one of main ones was how reliant my workflow had become on the `gh` CLI.<!--more-->

A colleague and myself have been working on our own Bitbucket PR CLI over at [quotidian-ennui/bitbucket-pr](https://github.com/quotidian-ennui/bitbucket-pr/), but thats another story.

My normal workflow in GitHub land is:

```shell
git co -b feat/cool
# do work
git aa
git cm "feat: something really cool"
git ph
gh pc
# Fill it out in browser
```

Which translated without `git` and `gh` aliases is:

```shell
git checkout --branch feat/cool
# do work
git add --all
git commit --message "feat: something really cool"
git push --set-upstream
gh pr create --web
# Fill it out in browser
```

The last part of the this workflow is the bit that's hard to replicate. Without taking your hand of the keyboard and clicking the link (lame).

Helpfully most git providers I've seen give the URL back in the response when you push, prefixed with `remote:` , which looks something like this:

```text
Total 0 (delta 0), reused 0 (delta 0), pack-reused 0
remote:
remote: Create a pull request for 'feat/something' on GitHub by visiting:
remote:      https://github.com/mcwarman/blog/pull/new/feat/something
remote:
To github.com:mcwarman/blog.git
 * [new branch]      feat/something -> feat/something
branch 'feat/something' set up to track 'origin/feat/something'.
```

So as the start to many problems I googled it and landed on this StackOverflow [post](https://stackoverflow.com/questions/42927782/extract-url-from-git-push-remote-response) about extracting the url from git push response.

Adapting that what I came up with is this, an awk that extracts the URL prints it to `.git/pull-request-url` as well as printing the full response.

```shell
git push --set-upstream --progress 2>&1 | awk '/^remote:.*https/ { print $2 > ".git/pull-request-url" }{ print }';
```

Then we can use `www-browser` (which for me is `wslview` under the covers):

```shell
www-browser $(cat .git/pull-request-url);
```

Now with the new commands we can turn them into git aliases by wrapping them in shell functions  `f() { ...; }; f` and using `!` as prefix which allows you run external commands in git alias:

```toml
# ~/.gitconfig
[alias]
  ph = "!f() { git push -u --progress 2>&1 | awk '/^remote:.*https/ { print $2 > \".git/pull-request-url\" }{ print }'; }; f"
  pr = "!f() { www-browser $(cat .git/pull-request-url); }; f"
```

The end of thew new workflow now looks like this, with the added bonus of working both on GitHub and Bitbucket.

```shell
git ph
git pr
# Fill it out in browser
```
