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
