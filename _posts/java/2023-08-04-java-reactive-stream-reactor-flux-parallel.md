--- 
layout: single
classes: wide
title: "[Java 실습] "
header:
  overlay_image: /img/java-bg.jpg 
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
toc: true 
use_math: true
---  

## Flux Parallel
`Flux Parallel` 이란 일반적으로 `Reactive Stream` 에서 `Producer` 는 시퀀스를 순차적으로 `next` 신호를 발생하고, 
이를 구독하는 `Subscriber` 는 순차적으로 신호를 처리하게 된다. 
하나의 `Producer` 의 `next` 신호를 병렬로 처리할 수 있는데, 
바로 `Flux` 의 `parallel()` 과 `runOn()` 을 사용하는 것이다.  

> 여기서 `parallel()` 과 `runOn()` 의 병렬처리와 `publishOn()` 과 `subscribeOn()` 의
> 병렬 처리의 목적은 구분이 필요하다. 
> `publishOn()` 과 `subscribeOn()` 에서 수행하는 병렬 처리는 각기 다른 `Reactive Stream` 을 병렬로 처리할 수 있도록 하는데 목적이 있다. 
> 이와 다르게 `parallel()` 과 `runOn()` 의 병렬처리는 하나의 `Reactive Stream` 에서 `downstream` 으로 보내는 `next` 신호를 벙렬로 처리하도록 하는데 목적이 있다.  
