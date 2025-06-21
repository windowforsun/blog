--- 
layout: single
classes: wide
title: "[LangChain] LangChain Document"
header:
  overlay_image: /img/langchain-bg-2.jpg
excerpt: 'LangChain 의 Document 와 Document Loader 에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - LangChain
tags:
    - Practice
    - LangChain
    - AI
    - LLM
    - Document
    - Document Loader
    - TextLoader
    - DirectoryLoader
    - JSONLoader
    - CSVLoader
    - DataFrameLoader
    - WebBaseLoader
toc: true
use_math: true
---  

## Document
`LangChain` 의 `Document` 는 `텍스트 조각` 과 그 텍스트에 대한 `메타데이터` 를 함께 담는 구조체를 의미한다. 
그러므로 이는 단순한 텍스트 문자열보다 더 많은 정보를 담을 수 있는 구조화된 형태이다. 

```python
from langchain_core.documents import Document

document = Document("나의 도큐먼트")

document.__dict__
# {'id': None, 'metadata': {}, 'page_content': '나의 도큐먼트', 'type': 'Document'}

document = Document(
    page_content="나의 도큐먼트",
    metadata={"source" : "note", "page": 1}
)

document.__dict__
# {'id': None,
# 'metadata': {'source': 'note', 'page': 1},
# 'page_content': '나의 도큐먼트',
# 'type': 'Document'}

document.metadata['author'] = 'windowforsun'

document.metadata
# {'source': 'note', 'page': 1, 'author': 'windowforsun'}
```  

위 예제와 같이 `Document` 은 아래와 같은 구조를 가지고 있다.  

- `page_content` : 문서 실제 내용(`text`)
- `metadata` : 문서와 관련된 추가 정보(`dict`)


