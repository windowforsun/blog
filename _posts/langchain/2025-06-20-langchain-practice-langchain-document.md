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



## Document Loader
`Document Loader` 는 다양한 파일의 형식으로부터 불러온 내용을 `Document` 객체로 변환하는 역할을 한다. 
주요한 `Document Loader` 에는 아래와 같은 것들이 있다. 
사용하 수 있는 `Document Loader` 는 [여기](https://python.langchain.com/docs/integrations/document_loaders/)
에서 확인할 수 있다. 

파일 기반 로더

- TextLoader: 일반 텍스트(.txt) 파일을 로드합니다.
- PyPDFLoader: PDF 파일을 로드하며, 페이지별로 Document 객체를 생성합니다.
- MergedPDFLoader: 여러 PDF를 하나의 Document로 병합하여 로드합니다.
- UnstructuredPDFLoader: 구조화되지 않은 PDF 파일에서 텍스트를 추출합니다.
- DocxLoader: Microsoft Word(.docx) 문서를 로드합니다.
- UnstructuredWordDocumentLoader: Word 문서(.doc, .docx)에서 텍스트를 추출합니다.
- CSVLoader: CSV 파일을 로드하며, 각 행을 별도의 Document로 변환할 수 있습니다.
- JSONLoader: JSON 파일을 로드하고 특정 경로의 데이터를 추출합니다.
- JSONLinesLoader: 각 줄이 JSON 객체인 파일을 로드합니다.
- UnstructuredExcelLoader: Excel 파일에서 텍스트를 추출합니다.
- UnstructuredEmailLoader: 이메일 파일(.eml, .msg)에서 텍스트를 추출합니다.
- UnstructuredMarkdownLoader: 마크다운 파일에서 텍스트를 추출합니다.
- UnstructuredHTMLLoader: HTML 파일에서 텍스트를 추출합니다.
- BSHTMLLoader: BeautifulSoup을 사용하여 HTML에서 텍스트를 추출합니다.

웹 기반 로더

- WebBaseLoader: 웹 URL에서 HTML 콘텐츠를 로드합니다.
- RecursiveUrlLoader: 웹사이트를 재귀적으로 크롤링하여 콘텐츠를 로드합니다.
- SitemapLoader: 사이트맵 XML 파일을 기반으로 웹사이트 콘텐츠를 로드합니다.
- ArxivLoader: Arxiv 논문을 검색하고 로드합니다.
- CollegeConfidentialLoader: College Confidential 포럼 스레드를 로드합니다.
- GitLoader: Git 저장소의 콘텐츠를 로드합니다.
- IFixitLoader: iFixit 가이드를 로드합니다.
- SeleniumURLLoader: Selenium을 사용하여 JavaScript가 필요한 웹페이지를 로드합니다.


데이터베이스 로더

- MongodbLoader: MongoDB 컬렉션에서 문서를 로드합니다.
- PostgresLoader: PostgreSQL 데이터베이스에서 데이터를 로드합니다.
- BigQueryLoader: Google BigQuery에서 데이터를 로드합니다.
- SqlDatabaseLoader: SQL 데이터베이스 테이블에서 데이터를 로드합니다.


기타 특수 로더

- DirectoryLoader: 디렉토리 내의 모든 파일을 로드합니다.
- UnstructuredFileLoader: 다양한 형식의 파일에서 텍스트를 추출합니다.
- S3DirectoryLoader: AWS S3 버킷에서 파일을 로드합니다.
- S3FileLoader: AWS S3 특정 파일을 로드합니다.
- BlockchainDocumentLoader: 블록체인 데이터를 로드합니다.
- ImageCaptionLoader: 이미지에서 캡션을 생성하여 로드합니다.
- AudioTranscriptionLoader: 오디오 파일을 텍스트로 변환하여 로드합니다.  

많은 `Document Loader` 중 몇 가지에 대해 간단히 살펴본다.  
