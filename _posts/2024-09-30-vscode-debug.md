---
layout: post
title: vscode自用调试配置
date: 2024-09-30
categories: blog
tags: [linux, vscode]
description: none
---

c++的

```json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
      {
        "name": "(gdb) Launch",
        "type": "cppdbg",
        "request": "launch",
        "program": "${workspaceFolder}/logrotate",
        "args": ["--debug","/etc/logrotate.conf"],
        "stopAtEntry": true,
        "cwd": "${workspaceFolder}",
        "environment": [],
        "externalConsole": false,
        "MIMode": "gdb",
        "setupCommands": [
          {
            "description": "Enable pretty-printing for gdb",
            "text": "-enable-pretty-printing",
            "ignoreFailures": true
          }
        ]
      }
    ]
  }
```

golang的

```json
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "xxx",
            "type": "go",
            "request": "launch",
            "mode": "auto",
            "program": "${workspaceFolder}/cmd/xxxx/main.go",
            "buildFlags": "-tags=dev",
            "env": {},
            "args": ["-config","example/config.yaml"],
            "showLog": true,
            "trace": "verbose",
            "cwd": "${workspaceFolder}"
        },
        {
            "name": "dynamic-main",
            "type": "go",
            "request": "launch",
            "mode": "auto",
            "program": "${file}",
            "buildFlags": "-tags=dev",
            "env": {},
            "args": [],
            "showLog": true,
            "trace": "verbose",
            "cwd": "${workspaceFolder}"
        },
    ]
}
```