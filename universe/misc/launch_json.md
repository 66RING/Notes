---
title: vscode launch.json
author: 66RING
date: 2000-01-01
tags: 
- debugger
- misc
mathjax: true
---

# vscode调试配置文件launch.json

cheat sheet

```json
{
   "$schema": "https://raw.githubusercontent.com/mfussenegger/dapconfig-schema/master/dapconfig-schema.json",
   "version": "0.2.0",
   "configurations": [
       {
           "type": "python",
           "request": "attach",
           "name": "docker attach",
           "pathMappings": [{
             "localRoot": "${workspaceFolder}",
             "remoteRoot": "."
           }],
           "justMyCode": false,
           "host": "127.0.0.1",
           "port": "9901"
       }
   ]
}
```


