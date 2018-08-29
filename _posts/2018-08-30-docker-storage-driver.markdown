---
layout: post
title:  "Docker storage driver"
date:   2018-08-30 12:00:00
author: Dark
categories: Docker
tags: docker
---


To use storage drivers effectively, it’s important to know how Docker builds and stores images, how these images are used by containers. You can use this information to make informed choices about the best way to persist data from your applications and avoid performance problems along the way.
Storage drivers allow you to create data in the writable layer of your container. The files won’t be persisted after the container stops, and both read and write speeds are low.
Learn how to use volumes to persist data and improve performance.

