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

## python

### attach

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

## cpp/gdb

### 直接调试二进制

```json
{
  "name": "Launch Local CUDA App",
  "type": "cppdbg",
  "request": "launch",
  "program": "${workspaceFolder}/build/my_app",
  "cwd": "${workspaceFolder}",
  "MIMode": "gdb",
  "miDebuggerPath": "/usr/bin/gdb",
  "setupCommands": [
    {
      "description": "Enable pretty-printing",
      "text": "-enable-pretty-printing",
      "ignoreFailures": true
    }
  ],
  "environment": [
    { "name": "CUDA_DEBUGGER_SOFTWARE_PREEMPTION", "value": "1" }
  ]
}
```

### attach

1. 启动server: `gdbserver :12345 ./my_app`

```json
{
    "name": "Attach to gdbserver :12345",
    "type": "cppdbg",
    "request": "launch",              // 注意：虽然是 attach，但仍用 launch
    "program": "${workspaceFolder}/build/my_app", // 与 gdbserver 启动的二进制必须一致
    "cwd": "${workspaceFolder}",
    "MIMode": "gdb",
    "miDebuggerServerAddress": "localhost:12345", // 远程填 IP:port
    "miDebuggerPath": "/usr/bin/gdb",  // 本地 gdb 路径
    "setupCommands": [
      {
        "description": "Enable pretty-printing for gdb",
        "text": "-enable-pretty-printing",
        "ignoreFailures": true
      }
    ],
    // 如果调试 CUDA 内核，需要额外设置
    "additionalSOLibSearchPath": "/usr/local/cuda/lib64",
    "environment": [
      { "name": "CUDA_DEBUGGER_SOFTWARE_PREEMPTION", "value": "1" }
    ]
}
```
