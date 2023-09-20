---
title: Mac卸载Docker
date: 2023-09-20 20:33:32
tags:
---

1. 运行下面命令卸载docker

`/Applications/Docker.app/Contents/MacOS/uninstall`

2. 删除应用程序中 docker.app
3. 删除以下目录

  ```shell
  rm -rf ~/Library/Containers/com.docker.docker
  rm -rf ~/Library/Group\ Containers/group.com.docker
  rm -rf ~/Library/Preferences/com.docker.docker.plist
  ```

4. 如需重装docker，需要重启电脑
