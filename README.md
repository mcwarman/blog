# blog

## Tasks

### new-post

Creating a new post.

Inputs: title

```shell
hugo new "posts/$(date "+%Y-%m-%d")-${title}.md"
```

### serve

Testing locally.

```shell
hugo serve
```
