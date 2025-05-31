--- 
layout: single
classes: wide
title: "[LangChain] LangChain Output Parsers"
header:
  overlay_image: /img/langchain-bg-2.jpg
excerpt: 'LangChain 에서 OutputParsers 를 사용해 LLM 의 출력을 구조화된 형태로 변환하는 방법을 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - LangChain
tags:
    - Practice
    - LangChain
    - AI
    - LLM
    - Prompt
    - Pydantic
    - OutputParsers
    - PydanticOutputParsers
    - CommaSeparatedListOutputParser
    - StructuredOutputParser
    - JsonOutputParser
    - PandasDataFrameOutputParser
    - DatetimeOutputParser
    - EnumOutputParser
    - OutputFixingParser
toc: true
use_math: true
---  

### OutputParsers
`OutputParsers` 는 `LangChain` 에서 언어 모델의 응답을 구조화된 형식으로 변환하는 도구이다. 
이를 통해 텍스트 형태의 `LLM` 출력을 프로그램 방식으로 처리할 수 있는 객체로 변환할 수 있다. 
`OutputParsers` 는 `LLM` 의 텍스트 출력을 구조화된 형식으로 변환하기 위해 아래와 같은 단계에서 사용된다. 

```
Prompt -> LLM -> Output -> OutputParsers -> Structured Output
```  

`OutputParsers` 의 주요 특징은 아래와 같다. 

- 구조화된 변환 : 자유 형식 텍스트를 `JSON`, 리스트, 사용자 정의 객체 등으로 변환
- 형식 지침 제공 : 언오 모델에게 어떤 형식으로 출력해야 하는지 안내하는 지침 생성
- 검증 기능 : 출력이 예상된 형식과 일치하는지 검증
- 에러 처리 : 파싱 실패 시 명확한 오류 메시지 제공
- 다양한 파서 유형 : 목적에 있는 여러 파서 제공(`JSON`, `Pydantic`, 리스트)

`OutputParsers` 의 주요 이점은 아래와 같다. 

- 일관성 보장 : 언어 모델의 구조와 형식을 일관되게 유지
- 후처리 간소화 : 텍스트 파싱에 드는 수고를 줄이고 코드 가독성 향상
- 통합 용이성 : 파싱된 데이터를 애플리케이션 로직에 쉽게 통합
- 프롬프트 최적화 : 형식 지침을 통해 원하는 출력 형태를 얻을 확률 증가
- 체인 구성 : 다른 `LangChain` 컴포넌트와 원활하게 연결 가능
- 타입 안정성 : 특히 `Pydantic` 파서 사용시 타입 검증을 통한 안정성 확보
- 재사용성 : 동일한 파서를 여러 `LLM` 요청에 재사용 가능
