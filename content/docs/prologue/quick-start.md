---
title: "Quick Start"
description: "How to run services."
lead: "How to run services."
date: 2020-11-16T13:59:39+01:00
lastmod: 2020-11-16T13:59:39+01:00
draft: false
images: []
menu:
  docs:
    parent: "prologue"
weight: 10
toc: true
---

## run

```bash
# run msg service
$ make run Srv=msg
# run gateway service
$ make run Srv=gateway
# run push service
$ make run Srv=push
```

## other make command

```bash
make help

Usage:
  make <target>

Development
  vet              Run go vet against code.
  lint             Run go lint against code.
  test             Run test against code.

Generate
  protoc           Run protoc command to generate pb code.

Build
  build            build provided server
  build-all        build all apps

Docker
  docker-build     build docker image

Run
  run              run provided server

General
  help             Display this help.
```
