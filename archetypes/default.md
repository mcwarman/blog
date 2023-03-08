---
title: {{ replace (replaceRE `\d{4}-\d{2}-\d{2}-(.*)` "$1" .Name) "-" " " | title }}
author: Matthew
date: {{ .Date }}
draft: true
---

