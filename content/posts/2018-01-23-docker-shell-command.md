---
title:  "Docker Shell Command"
author: "Matthew"
date: 2018-01-23
excerpt_separator: <!--more-->
---

After adding docker to my utility belt, I haven't looked back. One thing i often find myself needing to do is execute a bash session against my container.<!--more-->

What this looks like as docker command is:

```bash
docker exec -it <container> /bin/bash
```

In order to save key strokes i added it to my `~/.bashrc`.

```bash
docker-shell() {
  docker exec -it $1 /bin/bash
}
```

Now we have our command we can add tab completion for the container name argument:

```bash
_docker_shell()
{
  local procs
  _get_comp_words_by_ref -n : cur
  procs="$(docker ps -a --format '{{.Names}}')"
  COMPREPLY=( $(compgen -W "${procs}" -- ${cur}) )
  return 0
}

complete -F _docker_shell docker-shell
```

Again updating my `~/.bashrc` with the following line, where the part after the `.` is the location of the above script:

```bash
. /usr/share/bash-completion/completions/docker-shell
```

Now just a tab away from accessing the containers:

```bash
$ docker-shell m
mcwarman.github.io       mcwarman.github.io_blog  mysql
```

NOTE: With Git for Windows or Cygwin you'll prefix the `docker exec` with `winpty`.
