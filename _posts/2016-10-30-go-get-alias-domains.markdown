---
title: "Set up vanity go get aliases"
layout: post
date:   2016-10-30 00:00
description: "Setting up aliases for the go get command"
tags:
    - golang
    - goimport
    - import
---


> This guide was made from examples provided by the Kubernetes project, check it out [here](https://github.com/kubernetes/k8s.io/tree/master/k8s.io)!

A little-known fact of Go, is that you can use meta tags to redirect `go get` requests. This allows for custom alias domains when using public git providers (eg. GitHub, GitLab) which consequently makes it easier to maintain a stable identity for your package and ease the process of moving your repositories in future, as well as hosting repository mirrors.

When `go get` makes a request, it appends the `go-get=1` query parameter to the URL you specify, so `go get abc.xyz/example` will make a request to `http(s)://abc.xyz/example?go-get=1`. This page can specify meta tags that locate the repository to be retrieved.

An example response is as follows:

{% gist 17a7da41a2acda261b085bc33e937dee %}

The `go-import` tag is used to locate the repository for `go get`. We specify the `go-source` information for godoc support. More information on `go-source` is available [here](https://github.com/golang/gddo/wiki/Source-Code-Links)

If running on Kubernetes, here's a ready-made manifest for you to get started with!

{% gist de7c14f589beb7bd4549d86ba48b06d0 %}
