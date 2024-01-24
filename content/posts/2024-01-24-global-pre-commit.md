---
title: Global pre-commit
author: Matthew
date: 2024-01-24T13:11:11Z
---

I made a mistake when committing to repo recently I was using `terraform plan -generate-config-out=generate.tf` to bring some infrastructure under management of a repo. And managed to commit some secrets in the process by adding `generate.tf` by accident. I ended up nuking the repo because it was new enough that this was the easiest option.

But this got me thinking I should use `gitleaks` as `pre-commit` hook. And I wondered if I could do this globally. After throwing into google I found a few GitHub issues ([pre-commit#450](https://github.com/pre-commit/pre-commit/issues/450)) and stackoverflow answers ([ref](https://stackoverflow.com/questions/48301280/how-can-i-manually-run-a-git-pre-commit-hook-without-attempting-a-commit)). So bringing them together this is what I came up with.

Firstly I found a command that allows you to test `pre-commit` hooks, This was useful in testing `pre-commit` hooks.:

```shell
git hook run pre-commit
```

It's also possible to set a global hook directory in git:

```shell
git config --global core.hooksPath ~/.config/git/hooks
```

What this means is placing a `pre-commit` script into the `~/.config/git/hooks` directory it will be executed on every commit.

When you run `pre-commit install` in repo it creates `.git/hooks/pre-commit` so taking some inspiration from that I created new `pre-commit` script.

This will run global `pre-commit` config stored in `~/.config/pre-commit/pre-commit-config.yaml`, then run any `pre-commit` config in the repo.

### `~/.config/git/hooks/pre-commit`

```bash
#!/usr/bin/env bash

set -euo pipefail

args=(hook-impl --config="$HOME/.config/pre-commit/pre-commit-config.yaml" --hook-type=pre-commit)
hook_dir="$(cd "$(dirname "$0")" && pwd)"
args+=(--hook-dir "$hook_dir" -- "$@")

if type pre-commit &>/dev/null; then
  pre-commit "${args[@]}"
else
  echo 'pre-commit not found.' 1>&2
  exit 1
fi

if [ -e ./.git/hooks/pre-commit ]; then
  ./.git/hooks/pre-commit "$@"
else
  exit 0
fi
```

### `~/.config/pre-commit/pre-commit-config.yaml`

```yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.16.1
    hooks:
      - id: gitleaks-system
```

Testing product this, it ran `gitleaks` first, then the repos `terraform` hooks:

```text
Detect hardcoded secrets.................................................Passed
Terraform fmt............................................................Passed
Terraform validate with tflint...........................................Passed
Terraform docs...........................................................Passed
```

And re-testing how i failed it would have caught it:

```text
Detect hardcoded secrets.................................................Failed
- hook id: gitleaks
- exit code: 1

○
    │╲
    │ ○
    ○ ░
    ░    gitleaks

...REDACTED...

1:34PM INF 1 commits scanned.
1:34PM INF scan completed in 61.8ms
1:34PM WRN leaks found: 2
```
