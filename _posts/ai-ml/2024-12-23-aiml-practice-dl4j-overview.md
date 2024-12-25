--- 
layout: single
classes: wide
title: "[AI/ML] DL4J Overview"
header:
  overlay_image: /img/ai-ml-bg.png
excerpt: 'JVM 기반으로 분산 딥러닝 학습이 가능하고 다른 런타임과의 상호 운용성 및 대규모 데이터 처리가 가능한 DL4J 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - AI/ML
tags:
    - Practice
    - ML
    - DL4J
    - Java
    - DL
    - DeepLearning
toc: true
use_math: true
---

## DL4J
`Eclipse Deeplearning4j(DL4J`)는 `Java Virtual Machine(JVM)` 을 위한 오픈소스 분산 딥러닝 라이브러리이다. 
주로 `Java` 로 작성되었고 성능 최적화 를 위해 일부 네이티브 `C++` 컴포넌트를 포함하고 있다. 
`Java` 에서 모데를 학습시키는 동시에 `Python` 상태계와 상호작용할 수 있는 유일한 프레임워크로, 
`cpython` 바인딩을 통한 `Python` 실행 및 모델 가져오기 `tensorflow-java` 과 같은 다른 런타임과의 상호 운용성을 제공한다.  

또한 `Java` 나 `Scala` 를 사용해 분산 학습을 지원하고 `Apache Hadoop` 과 `Spark` 와 같은 빅데이터 프레임워크와 원할하게 통합되며, 
대규모 엔터프라이즈급 솔루션에 적합하다. 
이러한 `DL4J` 의 이점을 간략하게 정리하면 아래와 같다.  

- `JVM` 기반 : `Java` 및 `Scala` 로 작성되어 기존 `JVM` 애플리케이션과 쉽게 통합 가능하다. 
- `분산 학습` : 다수의 `GPU` 및 클러스터를 통한 대규모 학습을 지원한다. 
- `다양한 신경망 지원` : 피드포워드(`feed forward`), 순환 신경망(`RNN`), 합성곱 신경망(`CNN`) 등 다양한 신경망 아키텍쳐를 지원한다. 
- `빅데이터 도구와의 통합` : `Hadoop`, `Spark` 및 `Kubernetes` 와 함께 통합해 대규모 데이터 처리를 구현할 수 있다. 
- `맞춤화 가능` : 네트워크 구성 및 하이퍼파라미터 튜닝을 위한 유연성을 제공한다. 
- `호환성` : `Tensorflow`, `Keras` 와 같은 `Python` 라이브러리와의 `Import/Export` 기능으로 상호운용성을 제공한다. 


### Submodules
`DL4J` 는 다음과 같은 주요 구성 요소를 통해 구성된다.  

- `SameDiff` : 복잡한 그래프 실행을 위한 `Tensorflow/PyTorch` 와 유사한 프레임워크이다. 저수준이지만 유연하고, `onnx` 및 `Tensorflow` 그래프 실행을 위한 기본 `API` 이다. 
- `ND4J` : `Java` 용 `Numpy++` 로, `Numpy` 연산과 `Tensorflow/PyTorch` 연산의 조합을 포함한다. 
- `LibND4j` : 다양한 장치에서 수학 코드를 실행할 수 있도록 설계된 경량 독립형 `C++` 라이브러리다. 다양한 장치에서 실행 가능하도록 최적화할 수 있다. 
- `Python4J` : `Python` 스크립트의 프로덕션 배포를 간소화하는 `Python` 스크립트 실행 프레잌워크이다. 
- `Apache Spark Integration` : `Apache Spark` 프레이뭐크와 통합하여 `Spark` 에서 딥러닝 파이프라인 실행을 지원한다. 
- `DataVec` : 원시 입력 데이터를 신경망 실행에 적합한 텐서로 변환하는 데이터 변환 라이브러리이다. ETL(추출, 변환, 로드) 작업을 처리해 데이터 정규화, 특징 엔지니어링, 구조화된 데이터와 비구조화된 데이터를 처리할 수 있다. 
- `Model Zoo` : 이미지 인식, 자연어 처리(`NLP`)와 같은 일반적인 작업을 위한 사전 학습된 모델을 제공하여 모델 개발 시간을 절약할 수 있다. 
