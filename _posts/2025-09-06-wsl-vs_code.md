---
categories: Env
title: VS Code 与 WSL 配合使用
---
## 安装 VS Code 和 WSL 扩展

- 访问 [VS Code 安装页面](https://code.visualstudio.com/download)，选择适合当前系统体系结构的 **Windows 安装程序**。在 Windows 上安装 Visual Studio Code（而不是在 WSL 文件系统中）。
- 当在安装过程中提示“**选择其他任务”**时，请务必选中“**添加到 PATH**”选项，以便可以使用 code 命令轻松打开 WSL 中的文件夹。
- 安装[远程开发扩展包](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack)。除了远程 - SSH 和开发容器扩展之外，此扩展包还包括 WSL 扩展，使你能够打开容器、远程计算机或 WSL 中的任何文件夹。

## 在 Visual Studio Code 中打开 WSL 项目

![VS Code 的命令面板](https://learn.microsoft.com/en-us/windows/wsl/media/vscode-remote-command-palette.png)
