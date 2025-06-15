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

### PydanticOutputParsers
`PydanticOutputParsers` 는 `Pydantic` 모델기반의 구조화된 객체로 변환해 주는 파서이다. 
`Pydantic` 은 `Python` 의 데이터 검증 및 설정 관리 라이브러리로, 타입 어노테이션을 활용한 데이터 검증을 제공한다. 

작동 방식은 아래와 같다.

- `Pydantic` 모델 정의 : 원하는 출력 구조를 `Pydantic` 클래스로 정의
- 파서 생성 : 해당 모델을 기반으로 `PydanticOutputParsers` 인스턴스 생성
- 형식 지침 제공 : 파서가 `LLM` 에게 어떤 형식으로 응답해야 하는지 안내하는 지침 생성
- `LLM` 응답 파싱 : `LLM` 의 출력을 파싱하여 `Pydantic` 객체로 변환

주요 특징은 아래와 같다. 

- 타입 안전성 : `Python` 의 타입 시스템을 활용한 검증
- 자동 데이터 변환 : 문자열을 적절한 타입(정수, 날짜 등)으로 자동 변환
- 유효성 검증 : 필드 제약조건 및 사용자 정의 검증 로직 적용 가능
- 명확한 에러 메시지 : 파싱 실패 시 구체적인 오류 정보 제공
- API문서화 지원 : 자동 스키미 생성 기능 제공

`PydanticOutputParsers` 의 예제로 영화 리뷰를 분석하는 `Pydantic` 모델을 정의하고, 이를 활용해 `LLM` 응답을 파싱하는 방법을 살펴본다. 
먼저 비교를 위해 `OutputParsers` 를 사용하지 않는 경우를 살펴 보면 아래와 같다.  


```python
movie_review = """
올드보이 (Oldboy) 리뷰
감독: 박찬욱
개봉: 2003년 11월 21일
장르: 스릴러, 액션, 드라마

박찬욱 감독의 복수 3부작 중 두 번째 작품인 '올드보이'는 한국 영화사에 큰 획을 그은 걸작입니다.
15년간 이유도 모른 채 감금된 오대수(최민식)가 갑자기 풀려난 후 자신을 가둔 이유와 사람을 찾아가는 여정을 그립니다.
이 영화의 가장 큰 장점은 최민식의 압도적인 연기력입니다. 특히 유명한 복도 액션 시퀀스는 원테이크로 촬영되어 그 긴장감과
리얼리티가 극대화되었습니다. 또한 영화의 미장센과 색감 활용이 스토리텔링과 완벽하게 조화를 이루며 시각적 충격을 선사합니다.

스토리 측면에서는 복선과 반전이 절묘하게 배치되어 마지막까지 관객을 긴장시키며, 그리스 비극을 연상케 하는 
결말은 오랫동안 여운을 남깁니다. 영화 전반에 걸쳐 인간의 복수심과 구원에 대한 철학적 질문을 던지며 단순한 
복수극을 넘어선 깊이를 보여줍니다.

다만, 일부 잔인한 장면들과 충격적인 반전은 모든 관객에게 적합하지 않을 수 있습니다. 또한 서사의 복잡성으로 
인해 첫 관람에서는 모든 복선과 의미를 파악하기 어려울 수 있습니다.

영화는 2004년 칸 영화제에서 그랑프리를 수상했으며, 해외에서도 높은 평가를 받아 한국 영화의 세계화에 크게 
기여했습니다. 특히 쿠엔틴 타란티노 감독이 극찬한 작품으로도 유명합니다.

평점: 9.3/10
추천 대상: 심리 스릴러와 예술영화를 동시에 즐기고 싶은 성인 관객, 한국 영화의 예술성을 경험하고 싶은 해외 영화 팬
"""

from langchain_core.prompts import PromptTemplate

prompt = PromptTemplate.from_template("당신은 영화 리뷰 분석가입니다. 다음 영화 리뷰 내용중 중요한 내용을 한글로 추출해 주세요\n\n{movie_review}")

chain = prompt | model

answer = chain.invoke({"movie_review" : movie_review}
# 이 리뷰에서 중요한 내용은 다음과 같습니다.
# * 영화 '올드보이'는 한국 영화사에 큰 획을 그은 걸작으로, 박찬욱 감독의 복수 3부작 중 두 번째 작품입니다.
# * 이 영화의 가장 큰 장점은 최민식의 압도적인 연기력과 원테이크로 촬영된 복도 액션 시퀀스입니다.
# * 영화의 미장센과 색감 활용이 스토리텔링과 완벽하게 조화를 이루며 시각적 충격을 선사합니다.
# * 스토리 측면에서는 복선과 반전이 절묘하게 배치되어 마지막까지 관객을 긴장시키며, 그리스 비극을 연상케 하는 결말은 오랫동안 여운을 남깁니다.
# * 영화는 인간의 복수심과 구원에 대한 철학적 질문을 던지며, 단순한 복수극을 넘어선 깊이를 보여줍니다.
# * 일부 잔인한 장면들과 충격적인 반전은 모든 관객에게 적합하지 않을 수 있습니다.
# * 영화는 2004년 칸 영화제에서 그랑프리를 수상했으며,海外에서도 높은 평가를 받아 한국 영화의 세계화에 크게 기여했습니다.
# * 쿠엔틴 타란티노 감독이 극찬한 작품으로도 유명합니다.
# * 평점은 9.3/10이며, 심리 스릴러와 예술영화를 동시에 즐기고 싶은 성인 관객, 한국 영화의 예술성을 경험하고 싶은 해외 영화 팬에게 추천합니다.
```  

영화 리뷰 분석을 `PydnaticOutputParsers` 를 사용해 구조화된 형태로 변환하는 방법은 아래와 같다. 

```python
from typing import List, Optional
from pydantic import BaseModel, Field, validator
from langchain_core.output_parsers import PydanticOutputParser

class MovieReviewAnalysis(BaseModel):
    title_korean: str = Field(description="영화의 한글 제목")
    title_english: str = Field(description="영화의 영문 제목")
    director: str = Field(description="영화 감독의 이름")
    release_year: int = Field(description="영화 개봉 연도 (숫자만)")
    genres: List[str] = Field(description="영화 장르 목록")
    plot_summary: str = Field(description="영화의 줄거리 요약")
    pros: List[str] = Field(description="영화의 장점 목록")
    cons: List[str] = Field(description="영화의 단점 목록")
    rating: float = Field(description="영화 평점 (10점 만점)")
    recommendation: Optional[str] = Field(description="영화 추천 대상")

    @validator("rating")
    def validate_rating(cls, value):
        if value < 0 or value > 10:
            raise ValueError("평점은 0과 10 사이여야 합니다")
        return value

parser = PydanticOutputParser(pydantic_object=MovieReviewAnalysis)
parser.get_format_instructions()
# The output should be formatted as a JSON instance that conforms to the JSON schema below.
# 
# As an example, for the schema {"properties": {"foo": {"title": "Foo", "description": "a list of strings", "type": "array", "items": {"type": "string"}}}, "required": ["foo"]}
# the object {"foo": ["bar", "baz"]} is a well-formatted instance of the schema. The object {"properties": {"foo": ["bar", "baz"]}} is not well-formatted.
# 
# Here is the output schema:
# ```
# {"properties": {"title_korean": {"description": "영화의 한글 제목", "title": "Title Korean", "type": "string"}, "title_english": {"description": "영화의 영문 제목", "title": "Title English", "type": "string"}, "director": {"description": "영화 감독의 이름", "title": "Director", "type": "string"}, "release_year": {"description": "영화 개봉 연도 (숫자만)", "title": "Release Year", "type": "integer"}, "genres": {"description": "영화 장르 목록", "items": {"type": "string"}, "title": "Genres", "type": "array"}, "plot_summary": {"description": "영화의 줄거리 요약", "title": "Plot Summary", "type": "string"}, "pros": {"description": "영화의 장점 목록", "items": {"type": "string"}, "title": "Pros", "type": "array"}, "cons": {"description": "영화의 단점 목록", "items": {"type": "string"}, "title": "Cons", "type": "array"}, "rating": {"description": "영화 평점 (10점 만점)", "title": "Rating", "type": "number"}, "recommendation": {"anyOf": [{"type": "string"}, {"type": "null"}], "description": "영화 추천 대상", "title": "Recommendation"}}, "required": ["title_korean", "title_english", "director", "release_year", "genres", "plot_summary", "pros", "cons", "rating", "recommendation"]}
# ```

prompt = PromptTemplate.from_template(
    """
    당신은 영화 리뷰 분석가입니다. 다음 영화 리뷰를 분석하여 구조화된 정보로 변환해주세요:

    {review}

    결과는 요청된 아래 형식으로 정확히 제공해주세요.

    {format_instructions}
    """
)


prompt = prompt.partial(format_instructions=parser.get_format_instructions())
# PromptTemplate(input_variables=['review'], input_types={}, partial_variables={'format_instructions': 'The output should be formatted as a JSON instance that conforms to the JSON schema below.\n\nAs an example, for the schema {"properties": {"foo": {"title": "Foo", "description": "a list of strings", "type": "array", "items": {"type": "string"}}}, "required": ["foo"]}\nthe object {"foo": ["bar", "baz"]} is a well-formatted instance of the schema. The object {"properties": {"foo": ["bar", "baz"]}} is not well-formatted.\n\nHere is the output schema:\n```\n{"properties": {"title_korean": {"description": "영화의 한글 제목", "title": "Title Korean", "type": "string"}, "title_english": {"description": "영화의 영문 제목", "title": "Title English", "type": "string"}, "director": {"description": "영화 감독의 이름", "title": "Director", "type": "string"}, "release_year": {"description": "영화 개봉 연도 (숫자만)", "title": "Release Year", "type": "integer"}, "genres": {"description": "영화 장르 목록", "items": {"type": "string"}, "title": "Genres", "type": "array"}, "plot_summary": {"description": "영화의 줄거리 요약", "title": "Plot Summary", "type": "string"}, "pros": {"description": "영화의 장점 목록", "items": {"type": "string"}, "title": "Pros", "type": "array"}, "cons": {"description": "영화의 단점 목록", "items": {"type": "string"}, "title": "Cons", "type": "array"}, "rating": {"description": "영화 평점 (10점 만점)", "title": "Rating", "type": "number"}, "recommendation": {"anyOf": [{"type": "string"}, {"type": "null"}], "description": "영화 추천 대상", "title": "Recommendation"}}, "required": ["title_korean", "title_english", "director", "release_year", "genres", "plot_summary", "pros", "cons", "rating", "recommendation"]}\n```'}, template='\n    당신은 영화 리뷰 분석가입니다. 다음 영화 리뷰를 분석하여 구조화된 정보로 변환해주세요:\n\n    {review}\n\n    결과는 요청된 아래 형식으로 정확히 제공해주세요.\n\n    {format_instructions}\n    ')

chain = prompt | model

output = chain.invoke({'review' : movie_review}).content
# ```
# {
# "title_korean": "올드보이",
# "title_english": "Oldboy",
# "director": "박찬욱",
# "release_year": 2003,
# "genres": [
# "스릴러",
# "액션",
# "드라마"
# ],
# "plot_summary": "15년간 이유도 모른 채 감금된 오대수(최민식)가 갑자기 풀려난 후 자신을 가둔 이유와 사람을 찾아가는 여정을 그린 영화입니다.",
# "pros": [
# "최민식의 압도적인 연기력",
# "유명한 복도 액션 시퀀스의 원테이크 촬영",
# "영화의 미장센과 색감 활용이 스토리텔링과 조화를 이루며 시각적 충격을 선사함",
# "스토리 측면에서 복선과 반전이 절묘하게 배치되어 마지막까지 관객을 긴장시키는 결말"
# ],
# "cons": [
# "일부 잔인한 장면들과 충격적인 반전이 모든 관객에게 적합하지 않을 수 있음",
# "서사의 복잡성으로 인해 첫 관람에서는 모든 복선과 의미를 파악하기 어려울 수 있음"
# ],
# "rating": 9.3,
# "recommendation": "심리 스릴러와 예술영화를 동시에 즐기고 싶은 성인 관객, 한국 영화의 예술성을 경험하고 싶은 해외 영화 팬"
# }
# ```

structured_output = parser.parse(output)
# MovieReviewAnalysis(title_korean='올드보이', title_english='Oldboy', director='박찬욱', release_year=2003, genres=['스릴러', '액션', '드라마'], plot_summary='15년간 이유도 모른 채 감금된 오대수(최민식)가 갑자기 풀려난 후 자신을 가둔 이유와 사람을 찾아가는 여정을 그린 영화입니다.', pros=['최민식의 압도적인 연기력', '유명한 복도 액션 시퀀스의 원테이크 촬영', '영화의 미장센과 색감 활용이 스토리텔링과 조화를 이루며 시각적 충격을 선사함', '스토리 측면에서 복선과 반전이 절묘하게 배치되어 마지막까지 관객을 긴장시키는 결말'], cons=['일부 잔인한 장면들과 충격적인 반전이 모든 관객에게 적합하지 않을 수 있음', '서사의 복잡성으로 인해 첫 관람에서는 모든 복선과 의미를 파악하기 어려울 수 있음'], rating=9.3, recommendation='심리 스릴러와 예술영화를 동시에 즐기고 싶은 성인 관객, 한국 영화의 예술성을 경험하고 싶은 해외 영화 팬')
```  

`get_format_instructions()` 메서드를 통해 `Pydantic` 모델의 형식 지침을 확인할 수 있다. 
그리고 위 예제는 `LLM` 의 결과를 `parser()` 메서드를 사용해 `Pydantic` 객체로 변환한 예제이다. 
아래와 같이 체인에 `PydanticOutputParsers` 를 추가해 바로 변환된 객체를 얻을 수 있다. 

```python
chain = prompt | model | parser

output = chain.invoke({'review' : movie_review})
# MovieReviewAnalysis(title_korean='올드보이', title_english='Oldboy', director='박찬욱', release_year=2003, genres=['스릴러', '액션', '드라마'], plot_summary='15년간 이유도 모른 채 감금된 오대수가 갑자기 풀려난 후 자신을 가둔 이유와 사람을 찾아가는 여정을 그린 영화', pros=['최민식의 압도적인 연기력', '유명한 복도 액션 시퀀스의 원테이크 촬영', '영화의 미장센과 색감 활용이 스토리텔링과 완벽하게 조화를 이루며 시각적 충격을 선사', '스토리 측면에서 복선과 반전이 절묘하게 배치되어 마지막까지 관객을 긴장시키는 결말'], cons=['일부 잔인한 장면들과 충격적인 반전이 모든 관객에게 적합하지 않을 수 있음', '서사의 복잡성으로 인해 첫 관람에서는 모든 복선과 의미를 파악하기 어려울 수 있음'], rating=9.3, recommendation='심리 스릴러와 예술영화를 동시에 즐기고 싶은 성인 관객, 한국 영화의 예술성을 경험하고 싶은 해외 영화 팬')
```  

필요한 경우 아래와 같이 `with_structured_output()` 메서드를 사용해 출력 파서를 모델 객체에 직접 추가할 수도 있다.  

```python
model2 = init_chat_model("llama-3.3-70b-versatile", model_provider="groq").with_structured_output(MovieReviewAnalysis)

output = model2.invoke(movie_review)
# MovieReviewAnalysis(title_korean='올드보이', title_english='Oldboy', director='박찬욱', release_year=2003, genres=['스릴러', '액션', '드라마'], plot_summary='15년간 이유도 모른 채 감금된 오대수가 자신을 가둔 이유와 사람을 찾아가는 여정', pros=['최민식의 압도적인 연기력', '미장센과 색감 활용', '스토리텔링과 완벽한 조화'], cons=['일부 잔인한 장면과 충격적인 반전', '서사의 복잡성'], rating=9.3, recommendation='심리 스릴러와 예술영화를 동시에 즐기고 싶은 성인 관객, 한국 영화의 예술성을 경험하고 싶은海外 영화 팬')
```  


### CommaSeparatedListOutputParser
`CommaSeparatedListOutputParser` 는 출력을 쉼표로 구분된 항목들의 리스트로 변환하는 간단하지만 실용적인 파서이다. 
복잡한 구조가 필요하지 않고 단순히 여러 항목을 리스트 형태로 얻고 싶을 때 유용하다. 

작동 방식은 아래와 같다. 

- 텍스트 추출 : `LLM` 의 응답에서 텍스트를 가져온다. 
- 파싱 처리 : 텍스트를 쉽표를 구분자로 사용하여 분리한다. 
- 정제 작업 : 각 항목의 앞뒤 공백을 제거한다. 
- 리스트 생성 : 정제된 항목들을 `Python` 리스트로 반환한다. 

해당 파서의 장점은 다음과 같다. 

- 단순성 : 구현화 사용성이 매우 간단함
- 직관적 : `LLM` 에게 `쉽포로 구분된 목록을 생성해` 달라고 요청하는 것은 직관적임
- 가벼움 : 복잡한 파싱 로직이 필요하지 않음
- 실패 가능성이 낮음 : 단순한 형식으로 피상 실패 확률이 매우 낮음

단점은 아래와 같다. 

- 타입 제한 : 모든 항목이 문자열로 처리됨
- 복잡한 구조 불가 : 중첩 구조나 `키-값` 쌍을 표현할 수 없음
- 쉽표 포함 항목 처리 어려움 : 항목 자체에 쉼표가 포함된 경우 파싱이 복잡해짐
- 검증 부재 : 항목의 형식이나 유효성을 검증하지 않음

다음은 영화 리뷰를 분석해 주요 키워드 최대 5개를 추출하는 예제이다. 

```python
from langchain_core.output_parsers import CommaSeparatedListOutputParser

output_parser = CommaSeparatedListOutputParser()

format_instructions = output_parser.get_format_instructions()
# Your response should be a list of comma separated values, eg: `foo, bar, baz` or `foo,bar,baz`

prompt = PromptTemplate(
    template = """
    당신은 영화 리뷰 분석가입니다. 다음 영화 리뷰를 분석하여 주요 키워드 최대 5개를 추출해 나열하세요:

    {review}

    결과는 요청된 아래 형식으로 정확히 제공해주세요.

    {format_instructions}
    """,
    input_variables=['review'],
    partial_variables={'format_instructions' : format_instructions}
)

chain = prompt | model | output_parser

chain.invoke({'review' : movie_review})
# ['올드보이', '박찬욱', '복수', '스릴러', '최민식']
```  

### StructuredOutputParser
`StructuredOutputParser` 는 `PydanticOutputParser` 와 유사하지만, `Pydantic` 모델을 사용하지 않고 직접 구조화된 객체를 정의하는 파서이다. 
사전에 정의한 `카-값` 구조로 변환해주는 파서로 이해하면 쉽다. 
해당 파서는 `JSON` 형식의 출력을 생성하도록 `LLM` 에 지시하고, 이를 파싱하여 구조화된 딕셔너리 형태로 변환한다. 

작동 방식은 아래와 같다. 

- 구조 정의 : 원하는 출력 형식을 필드와 설명으로 정의
- 파서 생성 : 정의된 구조를 기반으로 `StructuredOutputParser` 인스턴스 생성
- 형식 지침 제공 : 파서가 `LLM` 에게 특정 `JSON` 구조로 응답하도록 안내하는 지침 제공
- `LLM` 응답 파싱 : `LLM` 의 출력을 파싱하여 `Python` 딕셔너리로 변환

주요 특징으로는 아래와 같은 것들이 있다. 

- 유연한 구조 정의 : 필드와 설명만으로 구조 정의 가능
- 간단한 사용법 : `Pydantic` 모델 정의 없이도 빠르게 구조화 가능
- 명확한 지침 생성 : `LLM` 에 제공할 명확한 형식 지침 자동 생성
- 중첩 구조 지춴 : 복잡한 중첩 `JSON` 구조도 정의 가능

`Pydantic/JSON` 파서가 좀 더 강력하지만, 해당 파서는 조금 더 간단한 모델(파라미터가 `GPT`, `Claude` 보다 적은) 에 유용하다.   


`StructuredOutputParser` 의 예제로 영화 리뷰를 분석해 출력 구조로 변환하는 방법을 살펴본다. 
이를 위해서는 스키마 정의가 필요한데, 
스키마 정의는 `JSON` 형식을 사용사거나 `ResponseSchema` 를 사용할 수 있다.  

```python
from langchain.output_parsers import ResponseSchema, StructuredOutputParser
from langchain_core.prompts import PromptTemplate

# 원하는 출력 구조 정의
response_schemas = [
    {
        "name": "title",
        "description": "영화의 제목"
    },
    {
        "name": "director",
        "description": "영화 감독의 이름"
    },
    {
        "name": "year",
        "description": "영화 개봉 연도 (숫자)"
    },
    {
        "name": "rating",
        "description": "영화 평점 (1-10 사이의 숫자)"
    },
    {
        "name": "summary",
        "description": "영화의 짧은 요약"
    },
    {
        "name": "review_summary",
        "description":"리뷰 짧은 요약"
    }
]
# response_schemas = [
#     ResponseSchema(name="title", description="영화의 제목"),
#     ResponseSchema(name="director", description="영화 감독의 이름"),
#     ResponseSchema(name="year", description="영화 개봉 연도 (숫자)"),
#     ResponseSchema(name="rating", description="영화 평점 (1-10 사이의 숫자)"),
#     ResponseSchema(name="summary", description="영화의 짧은 요약"),
#     ResponseSchema(name="review_summary", description="리뷰 짧은 요약")
# ]

# 파서 생성
parser = StructuredOutputParser.from_response_schemas(response_schemas)

# 형식 지침 확인
format_instructions = parser.get_format_instructions()
# The output should be a markdown code snippet formatted in the following schema, including the leading and trailing "```json" and "```":\n\n```json\n{\n\t"title": string  // 영화의 제목\n\t"director": string  // 영화 감독의 이름\n\t"year": string  // 영화 개봉 연도 (숫자)\n\t"rating": string  // 영화 평점 (1-10 사이의 숫자)\n\t"summary": string  // 영화의 짧은 요약\n}\n```

# 프롬프트 생성
prompt = PromptTemplate(
    template = """
    당신은 영화 리뷰 분석가입니다. 다음 영화 리뷰를 분석하여 구조화된 정보로 변환해주세요:

    {review}

    결과는 요청된 아래 형식으로 정확히 제공해주세요.

    {format_instructions}
    """,
    input_variables=['review'],
    partial_variables={'format_instructions' : format_instructions}
)

chain = prompt | model | parser
chain.invoke({'review' : movie_review})
# {'title': '올드보이 (Oldboy)',
#  'director': '박찬욱',
#  'year': '2003',
#  'rating': '9.3/10',
#  'summary': '15년간 이유도 모른 채 감금된 오대수(최민식)가 갑자기 풀려난 후 자신을 가둔 이유와 사람을 찾아가는 여정을 그린 영화',
#  'review_summary': '최민식의 압도적인 연기력, 원테이크 촬영의 긴장감, 영화의 미장센과 색감 활용이 스토리텔링과 조화를 이루며 시각적 충격을 선사하는 영화로,스토리 측면에서는 복선과 반전이 절묘하게 배치되어 마지막까지 관객을 긴장시키며, 그리스 비극을 연상케 하는 결말은 오랫동안 여운을 남깁니다.'}
```  

### JsonOutputParser
`JsonOutputParser` 는 `LLM` 의 텍스트 출력을 `JSON` 형식으로 파싱하는 간단하면서도 강력한 파서이다. 
해당 파서는 `LLM` 에게 `JSON` 형식으로 응답하도록 지시하고, 해당 응답을 `Python` 딕셔너리나 리스트로 변환한다. 

작동 방식은 아래와 같다.

- 형식 지침 제공: LLM에게 유효한 JSON 형식으로 응답하도록 안내하는 지침 생성
- 응답 수신: LLM으로부터 JSON 형식의 텍스트 응답 수신
- 마크다운 코드 블록 처리: JSON을 감싸고 있는 마크다운 코드 블록(````json`)이 있다면 제거
- 파싱 처리: 응답을 Python의 json.loads()를 사용하여 파싱
- 객체 변환: 파싱된 JSON을 Python 딕셔너리 또는 리스트로 변환

주요 특징으로는 아래와 같은 것들이 있다.

- 단순성: 별도의 스키마 정의나 Pydantic 모델 없이도 사용 가능
- 유연성: 동적으로 변화하는 JSON 구조도 처리 가능
- 자연스러운 지시: 대부분의 LLM은 JSON 형식 요청에 잘 반응함
- 중첩 구조 지원: 복잡한 중첩 JSON 객체도 쉽게 처리
- 오류 처리: JSON 파싱 오류 발생 시 명확한 에러 메시지 제공

`JsonOutputParser` 의 예제로 먼저 영화 리뷰를 분석해 형식에 맞게 출력하는 방법을 알아본다. 
먼저 `Pydantic` 모델을 사용하는 경우를 알아본다.  

```python
from typing import List, Optional
from pydantic import BaseModel, Field, validator
from langchain_core.output_parsers import JsonOutputParser


class MovieReviewAnalysis(BaseModel):
    title_korean: str = Field(description="영화의 한글 제목")
    title_english: str = Field(description="영화의 영문 제목")
    director: str = Field(description="영화 감독의 이름")
    release_year: int = Field(description="영화 개봉 연도 (숫자만)")
    genres: List[str] = Field(description="영화 장르 목록")
    plot_summary: str = Field(description="영화의 줄거리 요약")
    pros: List[str] = Field(description="영화의 장점 목록")
    cons: List[str] = Field(description="영화의 단점 목록")
    rating: float = Field(description="영화 평점 (10점 만점)")
    recommendation: Optional[str] = Field(description="영화 추천 대상")
    
    @validator("rating")
    def validate_rating(cls, value):
        if value < 0 or value > 10:
            raise ValueError("평점은 0과 10 사이여야 합니다")
        return value

parser = JsonOutputParser(pydantic_object=MovieReviewAnalysis)
# format_instructions = parser.get_format_instructions()
# The output should be formatted as a JSON instance that conforms to the JSON schema below.
# 
# As an example, for the schema {"properties": {"foo": {"title": "Foo", "description": "a list of strings", "type": "array", "items": {"type": "string"}}}, "required": ["foo"]}
# the object {"foo": ["bar", "baz"]} is a well-formatted instance of the schema. The object {"properties": {"foo": ["bar", "baz"]}} is not well-formatted.
# 
# Here is the output schema:
# ```
# {"properties": {"title_korean": {"description": "영화의 한글 제목", "title": "Title Korean", "type": "string"}, "title_english": {"description": "영화의 영문 제목", "title": "Title English", "type": "string"}, "director": {"description": "영화 감독의 이름", "title": "Director", "type": "string"}, "release_year": {"description": "영화 개봉 연도 (숫자만)", "title": "Release Year", "type": "integer"}, "genres": {"description": "영화 장르 목록", "items": {"type": "string"}, "title": "Genres", "type": "array"}, "plot_summary": {"description": "영화의 줄거리 요약", "title": "Plot Summary", "type": "string"}, "pros": {"description": "영화의 장점 목록", "items": {"type": "string"}, "title": "Pros", "type": "array"}, "cons": {"description": "영화의 단점 목록", "items": {"type": "string"}, "title": "Cons", "type": "array"}, "rating": {"description": "영화 평점 (10점 만점)", "title": "Rating", "type": "number"}, "recommendation": {"anyOf": [{"type": "string"}, {"type": "null"}], "description": "영화 추천 대상", "title": "Recommendation"}}, "required": ["title_korean", "title_english", "director", "release_year", "genres", "plot_summary", "pros", "cons", "rating", "recommendation"]}
# ```


prompt = PromptTemplate(
    template = """
    당신은 영화 리뷰 분석가입니다. 다음 영화 리뷰를 분석하여 구조화된 정보로 변환해주세요:

    {review}

    결과는 요청된 아래 형식으로 정확히 제공해주세요.

    {format_instructions}
    """,
    input_variables=['review'],
    partial_variables={'format_instructions' : format_instructions}
)

chain = prompt | model | parser

chain.invoke({'review' : movie_review})
# {'title_korean': '올드보이',
# 'title_english': 'Oldboy',
# 'director': '박찬욱',
# 'release_year': 2003,
# 'genres': ['스릴러', '액션', '드라마'],
# 'plot_summary': '15년간 이유도 모른 채 감금된 오대수가 갑자기 풀려난 후 자신을 가둔 이유와 사람을 찾아가는 여정을 그린 영화',
# 'pros': ['최민식의 압도적인 연기력',
#          '유명한 복도 액션 시퀀스의 긴장감과 리얼리티',
#          '영화의 미장센과 색감 활용이 스토리텔링과 완벽하게 조화를 이루며 시각적 충격을 선사하는 것'],
# 'cons': ['일부 잔인한 장면들과 충격적인 반전이 모든 관객에게 적합하지 않을 수 있음',
#          '서사의 복잡성으로 인해 첫 관람에서는 모든 복선과 의미를 파악하기 어려울 수 있음'],
# 'rating': 9.3,
# 'recommendation': '심리 스릴러와 예술영화를 동시에 즐기고 싶은 성인 관객, 한국 영화의 예술성을 경험하고 싶은 해외 영화 팬'}
```  

다음은 `Pydnatic` 을 사용하지 않는 경우이다. 
사용하지 않을 때는 `JSON` 을 응답하도록은 하지만, 스키마가 구체적으로 어떻게 되어야 하는지에 대한 구체적인 종보는 제공하지 않는다. 

```python
parser = JsonOutputParser()

prompt = PromptTemplate(
    template = """
    당신은 영화 리뷰 분석가입니다. 다음 영화 리뷰를 분석하여 JSON 형태의 구조화된 정보로 변환해주세요:

    {review}

    결과는 요청된 아래 필드를 정확히 제공해주세요.
    영화 제목은 `title`,
    영화 감독은 `director`,
    영화 개봉 연도는 `year`,
    영화 평점은 `rating`,
    영화 리뷰 요약은 `summary`
    """,
    input_variables=['review'],
)

chain = prompt | model | parser

chain.invoke({'review' : movie_review})
# {'title': '올드보이 (Oldboy)',
# 'director': '박찬욱',
# 'year': 2003,
# 'rating': 9.3,
# 'summary': "박찬욱 감독의 복수 3부작 중 두 번째 작품인 '올드보이'는 한국 영화사에 큰 획을 그은 걸작입니다. 15년간 이유도 모른 채 감금된 오대수(최민식)가 갑자기 풀려난 후 자신을 가둔 이유와 사람을 찾아가는 여정을 그립니다. 이 영화의 가장 큰 장점은 최민식의 압도적인 연기력입니다. 특히 유명한 복도 액션 시퀀스는 원테이크로 촬영되어 그 긴장감과 리얼리티가 극대화되었습니다. 또한 영화의 미장센과 색감 활용이 스토리텔링과 완벽하게 조화를 이루며 시각적 충격을 선사합니다."}
```  

### PandasDataFrameOutputParser
`PandasDataFrameOutputParser` 는 `LLM` 의 출력을 `Pandas` 의 `DataFrame` 으로 변환하는 전문화된 파서이다. 
`LLM` 에게 표 형태의 데이터를 요청하고, 반환된 결과를 바로 데이터 분석에 사용할 수 있는 `DataFrame` 으로 변환한다. 

작동 방식은 아래와 같다.

- 데이터 형식 지정 : 표 형태(`CSV` 등) 의 출력을 요청
- 텍스트 응답 수신 : `LLM` 으로부터 구조화된 텍스트 응답 수신
- 표 구조 인식 : 응답에서 표 구조 식별
- `DataFrame` 변환 : 파싱된 데이터를 `Pandas` 의 `DataFrame` 으로 구성
- 데이터 타입 추론 : 가능한 경우 열의 데이터 타입을 자동으로 추론

주요 특징으로는 아래와 같은 것들이 있다.

- 표 형태 데이터 처리 : 행과 열로 구성된 테이블 데이터에 최적화
- 데이터 분석 통합 : 파싱 즉싱 `Pandas` 의 강력한 분석 기능을 사용 가능
- 헤더 인식 : 테이블 헤더를 자동으로 인식하여 열 이름으로 설정
- 마크다운 표 지원 : 마크다운 형식의 표도 처리 가능
- 데이터 변환 : 텍스트 데이터를 적절한 숫자, 날짜 등으로 변환 시도 



`PandasDataFrameOutputParser` 의 예제로 주요 기업정보를 바탕으로 `DataFrame` 에 대해 분석하는 방법을 살펴본다. 

```python
import pandas as pd
from langchain.output_parsers import PandasDataFrameOutputParser
from langchain_core.prompts import PromptTemplate

data = {
    "Company": ["Apple", "Microsoft", "Amazon", "Google", "Tesla"],
    "Employees": [164000, 221000, 1608000, 190000, 127855],
    "Revenue": [387.5, 211.9, 524.9, 279.8, 81.5]  # 단위: billion USD
}
df = pd.DataFrame(data)

parser = PandasDataFrameOutputParser(dataframe=df)
# dataframe=     Company  Employees  Revenue
# 0      Apple     164000    387.5
# 1  Microsoft     221000    211.9
# 2     Amazon    1608000    524.9
# 3     Google     190000    279.8
# 4      Tesla     127855     81.5

prompt = PromptTemplate(
    template="쿼리에 대한 답을 찾아주세요. 최종 답변은 반드시 아래 포맷을 따르세요.\n{format_instructions}\n{query}\n",
    input_variables=["query"],
    partial_variables={"format_instructions": parser.get_format_instructions()},
)

chain = prompt | model | parser

output = chain.invoke({"query" : "첫 행을 조회해 주세요."})
# {'0': Company       Apple
#     Employees    164000
#      Revenue       387.5
#      Name: 0, dtype: object}
df.iloc[0]
# Company       Apple
# Employees    164000
# Revenue       387.5
# Name: 0, dtype: object

output = chain.invoke({"query" : "주어진 기업들의 평균 직원 수를 구해줘"})
# {'mean': np.float64(462171.0)}
df['Employees'].mean()
# 462171.0

output = chain.invoke({"query" : "주어진 기업들 중 가장 많은 직원을 가진 기업알려줘"})
# {'max': 1608000}
df['Employees'].max()
# 1608000

output = chain.invoke({"query" : "3번째까지 평균 매출 구해줘"})
# {'mean': np.float64(374.76666666666665)}
df['Revenue'].head(3).mean()
# 374.76666666666665

output = chain.invoke({"query" : "주어진 기업들의 총 수익 알려줘"})
# {'sum': np.float64(1485.6)}
df['Revenue'].sum()
# 1485.6
```  

사용해 봤을 때 느낀 점은 `LLM` 의 종류에 따라 활용도가 크게 달라질 것 같다. 
무료 모델로 사용했을 떄는 위와 같은 아주 간단한 통계식 질문의 답변을 얻을 수 있었고, 
복잡한 질문에 대해서는 답변을 얻기 어려웠다.  


### DatetimeOutputParser
`DatetimeOutputParser` 는 `LLM` 의 출력을 `Python` 의 `datetime` 객체로 변환하는 파서이다.

주요 특징으로는 아래와 같은 것들이 있다.

- 유연한 날짜 형식 인식 : 다양한 형식의 날짜/시간 표현을 인식
- 시간대 지원 : 시간대 정보가 포함된 `datetime` 객체 생성 가능
- 상대 시간 처리 : `내일`, `다음 주 금요일` 등의 상대적 표현 처리
- 예외 처리 : 잘못된 형식에 대한 명확한 오류 메시지 제공


아래는 `DatetimeOutputParser` 를 사용해서 날짜/시간에 대한 질문에 대한 답변을 얻는 예제이다. 

```python
from langchain.output_parsers import DatetimeOutputParser

parser = DatetimeOutputParser()
# 년 월 일까지만 결과에 표기하도록 설정
parser.format = '%Y-%m-%d'

prompt = PromptTemplate(
    template="사용자 질의에 다음 포맷에 맞게 답변하세요.: {query}\n\n{format_instructions}",
    input_variables=["query"],
    partial_variables={"format_instructions": parser.get_format_instructions()},
)
# PromptTemplate(input_variables=['query'], input_types={}, partial_variables={'format_instructions': "Write a datetime string that matches the following pattern: '%Y-%m-%d'.\n\nExamples: 1101-12-19, 1582-10-13, 1899-01-08\n\nReturn ONLY this string, no other words!"}, template='사용자 질의에 다음 포맷에 맞게 답변하세요.: {query}\n\n{format_instructions}')

chain = prompt | model | parser

output = chain.invoke({"query": "Apple 의 창업 연도는?"})
# datetime.datetime(1976, 4, 1, 0, 0)
output.strftime(parser.format)
# 1976-04-01

output = chain.invoke({"query": "빌게이츠의 생일은?"})
# datetime.datetime(1955, 10, 28, 0, 0)
output.strftime(parser.format)
# 1955-10-28

output = chain.invoke({"query": "오늘은 2002년 6월 10일 이야. 1년 하고도 3일 뒤 날짜 알려줘"})
# datetime.datetime(2003, 6, 13, 0, 0)
output.strftime(parser.format)
# 2003-06-13
```  

### EnumOutputParser
`EnumOutputParser` 는 `LLM` 의 출력을 `Python` 의 `Enum` 객체로 변환하는 특수 파서이다. 
정의된 `Enum` 의 값 중 하나로 매핑해 텍스트 응답을 제한된 옵션 세트로 제약할 수 있다. 

주요 특징으로는 아래와 같은 것들이 있다.

- 제한된 출력 볌위 : `LLM` 의 응답을 미리 정의된 값들로 제한
- 타입 안전성 : `Python` 의 `Enum` 타입을 활용하여 타입 안전성 확보
- 명확한 지침 제공 : `LLM` 에게 가능한 응답 옵션을 명확히 안내
- 대소문자 및 공백 처리 : 응답 텍스트를 정규화하여 매핑 성공률 향상
- 오류 처리 : 지정된 `Enum` 값과 일치하지 않는 경우 적절한 오류 메시지 제공 

아래는 `EnumOutputParser` 를 사용해서 문장의 감정을 분석하는 예제이다. 

```python
from langchain.output_parsers import EnumOutputParser
from enum import Enum

class Sentiment(Enum):
    POSITIVE = "긍정적"
    NEUTRAL = "중립적"
    NEGATIVE = "부정적"

parser = EnumOutputParser(enum=Sentiment)

prompt = PromptTemplate(
    template="사용자가 입력한 다음 문장의 감정을 분석하세요.: {query}\n\n최종 결과는 반드시 다음 출력 포맷:{format_instructions}",
    input_variables=["query"],
    partial_variables={"format_instructions": parser.get_format_instructions()},
)
# PromptTemplate(input_variables=['query'], input_types={}, partial_variables={'format_instructions': 'Select one of the following options: 긍정적, 중립적, 부정적'}, template='사용자가 입력한 다음 문장의 감정을 분석하세요.: {query}\n\n최종 결과는 반드시 다음

chain = prompt | model | parser

output = chain.invoke({"query" : "오늘 돈을 잃어버렸어"})
# <Sentiment.NEGATIVE: '부정적'>

output = chain.invoke({"query" : "오늘 좋아하는 여자와 밥을 먹었어"})
# <Sentiment.POSITIVE: '긍정적'>

output = chain.invoke({"query" : "이런저런 일들이 많은 하루였어"})
# output = chain.invoke({"query" : "이런저런 일들이 많은 하루였어"})
```  


### OutputFixingParser
`OutputFixingParser` 는 `LLM` 의 특수한 래퍼 파서로, 
다른 파서가 실패할 때 자동으로 문제를 해결하는 역할을 한다. 
이는 파서 식패 복구 매커니즘으로 작동하며, 
원래 파서가 `LLM` 응답을 처리하지 못할 때 오류를 `LLM` 에게 다시 전달하여 올바른 형식으로 수정하도록 요청하는 방식이다. 

작동 방식은 아래와 같다. 

- 기본 파싱 시도 : 우선 래핑된 원본 파서로 `LLM` 출력을 파싱
- 오류 감지 : 파싱 실패 시 오류 메시지 캡쳐
- 오류 설명 및 재시도 : 오류와 함께 원본 출력을 `LLM` 에 다시 전달
- 수정된 출력 : `LLM` 이 올바른 형식으로 응답을 수정
- 재파싱 : 수정된 응답으로 다시 원본 파서 실행

주요 특징으로는 아래와 같은 것들이 있다.

- 자동 오류 복구 : 파서 오류를 자동으로 감지하고 해결
- 형식 오류 수정 : `JSON` 문법 오류, 날짜 형식 오류 등 자동 수정
- 사용자 경험 향샹 : 최종 사용자에게 오류 대신 올바른 결과 제공 가능
- 개발 부담 감소 : 엄격한 파싱 로직을 유지하면서도 실패 처리 자동화
- 적응형 파싱 : 다양한 `LLM` 응답 변형에 유연하게 대응

아래는 `OutputFixingParser` 의 몇 사용 예제이다. 

```python
from langchain_core.output_parsers import JsonOutputParser
from langchain.output_parsers import OutputFixingParser

json_parser = JsonOutputParser()

fixing_parser = OutputFixingParser.from_llm(parser=json_parser, llm=model)

# " 와 , 누락
incorrect_json = """
{
  name: 철수,
  age: 20
  job: "공무원"
}
"""

try:
  json_parser.parse(incorrect_json)
except Exception as e:
  print(e)
# Invalid json output: {
#     name: 철수,
#     age: 20
#     job: "공무원"
# }
# For troubleshooting, visit: https://python.langchain.com/docs/troubleshooting/errors/OUTPUT_PARSING_FAILURE

fixing_parser.parse(incorrect_json)
# {'name': '철수', 'age': 20, 'job': '공무원'}


# { 와 " 그리고 , 누락
incorrect_json = """

  name: 철수,
  age: 20
  job: "공무원"

"""

try:
    json_parser.parse(incorrect_json)
except Exception as e:
    print(e)
# Invalid json output: name: 철수,
# age: 20
# job: "공무원"
# For troubleshooting, visit: https://python.langchain.com/docs/troubleshooting/errors/OUTPUT_PARSING_FAILURE

fixing_parser.parse(incorrect_json)
# {'name': '철수', 'age': 20, 'job': '공무원'}



class Employee(BaseModel):
    id: int = Field(description="직원 번호")
    name: str = Field(description="직원 이름")
    projects: List[str] = Field(description="참여 중 인 1개 이상")


model_json_parser = JsonOutputParser(pydantic_object=Employee)

model_json_fixing_parser = OutputFixingParser.from_llm(parser=model_json_parser, llm=model)

# " 와 , 누락 및 배열 타입
incorrect_json = """
{
  id : 1
  name : 철수
  projects : "AI, DEV"
}
"""



try:
    model_json_parser.parse(incorrect_json)
except Exception as e:
    print(e)
# Invalid json output: {
#     id : 1
#     name : 철수
#     projects : "AI, DEV"
# }
# For troubleshooting, visit: https://python.langchain.com/docs/troubleshooting/errors/OUTPUT_PARSING_FAILURE

model_json_fixing_parser.parse(incorrect_json)
# {'id': 1, 'name': '철수', 'projects': ['AI', 'DEV']}



# { 와 " 그리고 , 누락 및 배열 타입
incorrect_json = """

  id : 1
  name : 철수
  projects : "AI", "DEV"

"""


try:
    model_json_parser.parse(incorrect_json)
except Exception as e:
    print(e)
# Invalid json output: id : 1
# name : 철수
# projects : "AI", "DEV"
# For troubleshooting, visit: https://python.langchain.com/docs/troubleshooting/errors/OUTPUT_PARSING_FAILURE

model_json_fixing_parser.parse(incorrect_json)
# {'id': 1, 'name': '철수', 'projects': ['AI', 'DEV']}
```



---  
## Reference
[Output parsers](https://python.langchain.com/docs/concepts/output_parsers/)  
[출력파서(Output Parser)](https://wikidocs.net/233771)  




