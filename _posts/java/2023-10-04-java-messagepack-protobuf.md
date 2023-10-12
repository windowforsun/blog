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
  - MessagePack
  - Protobuf
  - Serialize
  - Deserialize
toc: true 
use_math: true
---  

## Serialize
`Serialize` 는 프로그램이 사용하는 객체를 `Data stream` 으로 만드는 것을 의미한다. 
반대로 `Data stream` 을 다시 프로그램이 사용할 수 있는 객체로 변환하는 것을 `Deserialize` 라고 한다.  

이러한 직렬화와 역직렬화가 필요한 이유는 객체 자체의 영속적으로 보관할 때 파일형대태로 만들어 저장소에 저장하거나, 
네트워크를 통해 다른 엔드포인트로 전달이 가능하다.  

직렬화에는 많은 방식들이 있지만, 서버-클라이언트 사이의 데이터 송수신에는 주로 `Json` 이 사용된다. 
`Json` 도 `HTML`, `XML` 등을 거치면서 단점을 보완한 직렬화 포멧이지만, 
이 또한 데이터 크기나 직렬화 퍼포먼스등의 단점이 존재한다.  

그래서 이번 포스트에서는 `Java` 언에서 `Json` 을 대신해서 사용할 수 있는 직렬화 포멧인 `MessagePack` 과 `Protobuf` 에 대해 알아본다.  
