---
title: Everything could be Code
author: Matthew
date: 2024-01-19T11:39:43Z
draft: true
---

Everything as Code (EaC) is far from a new concept, and anyone in a platform or devops role has embraced this is mantra sometimes unknowingly.

The concept is easy for new resources, new initiatives and new projects. But we live in a world were we've either adopted a project that was "Click-Ops'd" or we were the engineer doing the clicking.

If Everything as Code is the question, terraform is normally the answer.

But the real question is how do we make that `resource` code that was set-up a long time a go and no one really knows how it was set-up.

The answer previously was you'd open up the terraform docs of that resource and meticulously go through the settings with the cloud console open on another screen. Import the resource via CLI in to your state, tell all other engineers that this repo cannot be touched until you've finished and see what you'd break if you ran the plan. Update the settings, re-run plan, repeat.

This process usually means the task is never undertaken as more important this will inevitably come up.

With Terraform version 1.5 two features became available `import` block, which allows an engineer to show an intent of updating state with you making the change. And an experimental feature that enables the [generation of code from resources](https://developer.hashicorp.com/terraform/language/import/generating-configuration).

This post is my attempt of utilising these features, to bring some infrastructure into management, so we can roll out new settings to production (hopefully the correct way).

The goal is to create a project that can manage an existing Azure App Service.
