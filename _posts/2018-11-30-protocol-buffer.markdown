---
layout: post
title:  "protocol buffer"
date:   2018-11-30 01:00:00
author: Dark
categories: protocol buffer
tags: protocol buffer
---

이 튜토리얼은 기본적인 Java programmer 들이 gRPC 를 사용하는데 기본적으로 알아야할 사항들에 대해 가이드한다.

이 예제를 살펴보면 다음과 같은 사항들을 배우게 된다:

- service 를 .proto file 에 정의할 수 있다.
- Server 와 Client 코드를 protocol buffer compiler 를 사용하여 생성할 수 있다.
- Java gRPC API 를 간단한 client 와 server 를 생성하는데 사용할 수 있다.

이 가이드는 당신이 이미 Overview 를 읽었고, protocol buffer 에 친숙하다는 가정하에 작성되었다.
이 튜토리얼의 예제는 proto3 버전의 protocol buffer 언어를 사용합니다:
proto3 language 가이드와 Java generated code 가이드에서 자세한 내용을 확인할 수 있습니다.
또한, protocol buffers GitHub repository 에서 새로운 버전에 대한 release notes 를 확인할 수 있습니다.

### Reference
[protocol buffer]

[protocol buffer]:      https://developers.google.com/protocol-buffers/docs/overview