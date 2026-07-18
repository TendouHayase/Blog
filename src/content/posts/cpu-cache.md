---
title: CPU 캐시 구조와 접근, 갱신
published: 2026-07-05
description: '캐시 주소 구조 및 캐시 갱신 과정'
image: ''
tags: ["Computer Science"]
category: 'Knowledge'
draft: true
lang: 'ko'
---

## 메모리 주소와 CPU 캐시 매핑

내용을 설명하기 앞서 cpu는 기본적으로 메모리에서 데이터를 가져올 때 캐시 라인 크기 단위(Ryzen 9700x 기준 64바이트)로 데이터를 가져옵니다.
따라서 cpu 캐시의 주소 해석 또한 이 캐시 라인 크기 단위로 이루어집니다.
