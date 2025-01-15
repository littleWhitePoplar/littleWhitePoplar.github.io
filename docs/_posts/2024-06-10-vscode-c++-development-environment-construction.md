---
layout: post
title: "c++开发环境搭建"
date: 2024-06-10 08:00:00 +0000
categories: jekyll update
---

## 插件安装

> 如果是windows平台，仅需安装c/c++插件即可

1. clangd  
    clangd是一种包含代码补全，代码跳转，错误提示等功能的语言服务，它支持在许多编辑器中作为插件来使用。  
    1. 在vscode的扩展中搜索并安装clangd
    2. 在linux中安装clangd  
        ```bash
        sudo apt install clangd
        ```  
    3. 生成项目的compile_commands.json  
        compile_commands.json是编译命令的数据库，clangd需要依赖于它实现相关功能，通常它位于项目的build目录下。  
        对于cmake，我们使用以下脚本来生成
        ```bash
        cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON ....
        ```
        对于make,scons等其他的脚本，我们使用以下脚本来生成
        ```bash
        # for make
        bear make ....
        # for scons
        bear soncs ....
        # for other
        bear ....
        ```
        > 注意：使用bear生产等的compile_commands.json在当前目录下，需要手动拷贝到项目的build目录下
2. clang-format  
    clang-format是一种用来格式化源码的插件，它支持C/C++/Java/JavaScript等很多语言。它通过项目目录下的.clang-format文件来配置格式化。
    1. 在vscode的扩展中搜索并安装clang-format
    2. 在linux中安装clang-format
        ```bash
        sudo apt install clang-format
        ```
    3. 生成默认的配置模板到当前项目
       ```bash
       # style包含llvm,chromium,google等
       clang-format -style=google -dump-config > .clang-format
       ```
3. clang-tidy  
    clang-tidy是一个基于clang的C++“linter”工具。它可以检查到代码中存在的问题并自动修复。它依赖于项目中的compile_commands.json来实现相关功能。
    1. 在vscode的扩展中搜索并安装clang-tidy
    2. 在linux中安装clang-tidy
        ```bash
        sudo apt install clang-tidy
        ```
    3. 修改clang-tidy的扩展设置，因为它默认compile_command.json在当前目录下，需要修改为build/ 即可。
4. c/c++  
    该插件提供了IntellSense，debug的功能，这里我们仅需要它的debug功能。  
    1. 在vscode扩展中搜索并安装c/c++
    2. 在扩展设置中关闭IntellSense功能

## vscode配置

为了支撑c++的开发，我们需要在vscode中配置以下两个配置文件。

1. .vscode/tasks.json  

tasks.json 文件用于配置任务运行器。我们可以在这里定义可以通过任务运行器执行的任务，例如编译代码、运行测试等。

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "echo",
            "type": "shell",
            "command": "echo Hello, World",
            "problemMatcher": []
        }
    ]
}
```

2. .vscode/launch.json  

launch.json 文件用于配置调试器。我们可以定义不同的调试配置，以便在 VS Code 中调试应用程序  

```json
{
  "name": "C++ Launch",
  "type": "cppdbg",
  "request": "launch",
  "program": "${workspaceFolder}/a.out",
  "args": ["arg1", "arg2"],
  "environment": [{ "name": "config", "value": "Debug" }],
  "cwd": "${workspaceFolder}"
}
```

