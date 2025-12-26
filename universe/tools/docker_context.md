---
title: docker context usage
author: 66RING
date: 2025-12-26
tags: 
- docker
- tools
mathjax: true
---

# Docker context用例

把一个远程机器的docker映射到本地docker。这样一来, 本地docker ps就相当于在远程机器docker ps

```bash
docker context create some-context-label --docker "host=ssh://user@remote_server_ip"

docker context use some-context-label

docker ps
```

这时, vscode连接时就从原来的vscode -> ssh remote -> remote docker变成了vscode -> remote docker。不再需要端口转发。
