---
title: STL의 vector 파헤치기
published: 2026-09-11
description: 'msvc STL에 있는 vector가 어떻게 동작하는지 살펴보기'
image: ''
tags: ["C++", "STL"]
category: ''
draft: true 
lang: 'ko'
---

## 서론

이 글은 기본적으로 `SFINAE`(Substitution Failure Is Not An Error)의 개념을 알고있다는 가정하에 작성되었습니다.

우리가 `std::vector<T>`를 사용할 때 내부에서 어떻게 동작하는지 할당부터 `[ ]`과 `pop_back()`을 통해 알아볼 것 입니다.

이 글을 쓰기 위해 살펴볼 때 사용한 MSVC는 MSVC 2022 14.44.35207 버전입니다.

## vector 클래스

먼저 저희가 `std::vector`를 쓸 때는 `std::vector`에 저장할 자료형 1개를 템플릿 인자로 주어 생성합니다.

따라서 `std::vector`의 실제 템플릿 매개변수도 1개라 생각할 수도 있는데 실제로는 2개고 두번째 템플릿 인자는 기본적으론 지정되어져있습니다.
