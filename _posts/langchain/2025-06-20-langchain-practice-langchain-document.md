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

### TextLoader
`TextLoader` 는 가장 기본적인 문서 로더로 일반 텍스트(`.txt`) 파일을 읽어 `Document` 로 반환하는 역할을 한다. 
텍스트 파일 내용을 읽어서 `Document` 객체의 `page_content` 필드에 저장하고, 
파일 경로 등의 정보는 `metadata` 에 저장한다. 

사용 가능한 주요 매개변수는 아래와 같다. 

```python
TextLoader(
    # 로드할 텍스트 파일 경로
    file_path: str,
    # 파일을 읽을 때 사용할 인코딩 형식, 기본값은 None(시스템 기본 인코딩 사용)
    encoding: Optional[str] = None,
    # 인코딩을 자동으로 감지할지 여부 (기본값: False)
    autodetect_encoding: bool = False
)
```  


아래와 같은 영화관련 정보가 담긴 `TXT` 파일을 로드해 사용한다.


<details><summary>JSON 내용</summary>
<div markdown="1">

```txt
Movies:
1. Title: Inception
   Director: Christopher Nolan
   Release Year: 2010
   Genre: Action, Sci-Fi, Thriller
   Rating: 8.8
   Cast:
    - Leonardo DiCaprio as Dom Cobb
    - Joseph Gordon-Levitt as Arthur
    - Elliot Page as Ariadne

2. Title: The Shawshank Redemption
   Director: Frank Darabont
   Release Year: 1994
   Genre: Drama
   Rating: 9.3
   Cast:
    - Tim Robbins as Andy Dufresne
    - Morgan Freeman as Ellis Boyd 'Red' Redding

3. Title: The Matrix
   Director: The Wachowskis
   Release Year: 1999
   Genre: Action, Sci-Fi
   Rating: 8.7
   Cast:
    - Keanu Reeves as Neo
    - Laurence Fishburne as Morpheus
    - Carrie-Anne Moss as Trinity

4. Title: Parasite
   Director: Bong Joon-ho
   Release Year: 2019
   Genre: Drama, Thriller
   Rating: 8.5
   Cast:
    - Song Kang-ho as Kim Ki-taek
    - Lee Sun-kyun as Park Dong-ik
    - Cho Yeo-jeong as Park Yeon-kyo

5. Title: Avengers: Endgame
   Director: Anthony Russo, Joe Russo
   Release Year: 2019
   Genre: Action, Adventure, Sci-Fi
   Rating: 8.4
   Cast:
    - Robert Downey Jr. as Tony Stark / Iron Man
    - Chris Evans as Steve Rogers / Captain America
    - Scarlett Johansson as Natasha Romanoff / Black Widow

6. Title: Spirited Away
   Director: Hayao Miyazaki
   Release Year: 2001
   Genre: Animation, Adventure, Fantasy
   Rating: 8.6
   Cast:
    - Rumi Hiiragi as Chihiro Ogino (voice)
    - Miyu Irino as Haku (voice)
    - Mari Natsuki as Yubaba / Zeniba (voice)

7. Title: The Godfather
   Director: Francis Ford Coppola
   Release Year: 1972
   Genre: Crime, Drama
   Rating: 9.2
   Cast:
    - Marlon Brando as Don Vito Corleone
    - Al Pacino as Michael Corleone
    - James Caan as Sonny Corleone

8. Title: Interstellar
   Director: Christopher Nolan
   Release Year: 2014
   Genre: Adventure, Drama, Sci-Fi
   Rating: 8.6
   Cast:
    - Matthew McConaughey as Cooper
    - Anne Hathaway as Brand
    - Jessica Chastain as Murph
```  

</div>
</details>  


```python
from langchain_community.document_loaders import TextLoader

loader = TextLoader("./movies.txt")

docs = loader.load()
# [Document(metadata={'source': './movies.txt'}, page_content="Movies:\n1. Title: Inception\n   Director: Christopher Nolan\n   Release Year: 2010\n   Genre: Action, Sci-Fi, Thriller\n   Rating: 8.8\n   Cast:\n     - Leonardo DiCaprio as Dom Cobb\n     - Joseph Gordon-Levitt as Arthur\n     - Elliot Page as Ariadne\n\n2. Title: The Shawshank Redemption\n   Director: Frank Darabont\n   Release Year: 1994\n   Genre: Drama\n   Rating: 9.3\n   Cast:\n     - Tim Robbins as Andy Dufresne\n     - Morgan Freeman as Ellis Boyd 'Red' Redding\n\n3. Title: The Matrix\n   Director: The Wachowskis\n   Release Year: 1999\n   Genre: Action, Sci-Fi\n   Rating: 8.7\n   Cast:\n     - Keanu Reeves as Neo\n     - Laurence Fishburne as Morpheus\n     - Carrie-Anne Moss as Trinity\n\n4. Title: Parasite\n   Director: Bong Joon-ho\n   Release Year: 2019\n   Genre: Drama, Thriller\n   Rating: 8.5\n   Cast:\n     - Song Kang-ho as Kim Ki-taek\n     - Lee Sun-kyun as Park Dong-ik\n     - Cho Yeo-jeong as Park Yeon-kyo\n\n5. Title: Avengers: Endgame\n   Director: Anthony Russo, Joe Russo\n   Release Year: 2019\n   Genre: Action, Adventure, Sci-Fi\n   Rating: 8.4\n   Cast:\n     - Robert Downey Jr. as Tony Stark / Iron Man\n     - Chris Evans as Steve Rogers / Captain America\n     - Scarlett Johansson as Natasha Romanoff / Black Widow\n\n6. Title: Spirited Away\n   Director: Hayao Miyazaki\n   Release Year: 2001\n   Genre: Animation, Adventure, Fantasy\n   Rating: 8.6\n   Cast:\n     - Rumi Hiiragi as Chihiro Ogino (voice)\n     - Miyu Irino as Haku (voice)\n     - Mari Natsuki as Yubaba / Zeniba (voice)\n\n7. Title: The Godfather\n   Director: Francis Ford Coppola\n   Release Year: 1972\n   Genre: Crime, Drama\n   Rating: 9.2\n   Cast:\n     - Marlon Brando as Don Vito Corleone\n     - Al Pacino as Michael Corleone\n     - James Caan as Sonny Corleone\n\n8. Title: Interstellar\n   Director: Christopher Nolan\n   Release Year: 2014\n   Genre: Adventure, Drama, Sci-Fi\n   Rating: 8.6\n   Cast:\n     - Matthew McConaughey as Cooper\n     - Anne Hathaway as Brand\n     - Jessica Chastain as Murph")]
```  


### DirectoryLoader
`DirectoryLoader` 는 디렉토리 내의 여러 파일을 한 번에 로드하여 `Document` 객체로 변환하는 역할을 한다. 
디렉토리에 있는 파일들을 필터링하고, 지정된 로드를 사용하여 각 파일을 `Document` 객체로 변환한다. 

주요 매개변수는 아래와 같다.  


```python
DirectoryLoader(
    # 로드할 파일이 있는 디렉토리 경로
    path: str,
    # 파일 필터링을 위한 패턴, 기본값 **/*
    glob: str = "**/*",
    # 각 파일을 로드하는데 사용할 로더 클래스
    loader_cls: Optional[Type[BaseLoader]] = None,
    # 로더 클래스에 전달할 추가 인자
    loader_kwargs: Optional[Dict[str, Any]] = None,
    # 하위 디렉토리까지 검색할지 여부, 기분값 False
    recursive: bool = False,
    # 진행 상황 표시 여부, 기본값 False
    show_progress: bool = False,
    # 멀티스레딩 사용 여부, 기본값 False
    use_multithreading: bool = False,
    # 최대 동시 실행 스레드 수
    max_concurrency: Optional[int] = None,
    # 오류 발생 시 건너뛸지 여부, 기본값 False
    silent_errors: bool = False
)
```  

예제는 앞서 `TextLoader` 에서 사용한 영화에 대한 내용을 `movies` 라는 디렉토리에 각 영화별로 분리해 저장해 진행한다.  


```python
from langchain_community.document_loaders import DirectoryLoader

loader = DirectoryLoader(
    path='./movies/', 
    glob="**/*.txt", 
    loader_cls=TextLoader,
    loader_kwargs = {'autodetect_encoding': True}
)
docs = loader.load()
# [Document(metadata={'source': 'movies/4.txt'}, page_content='Title: Parasite\n   Director: Bong Joon-ho\n   Release Year: 2019\n   Genre: Drama, Thriller\n   Rating: 8.5\n   Cast:\n     - Song Kang-ho as Kim Ki-taek\n     - Lee Sun-kyun as Park Dong-ik\n     - Cho Yeo-jeong as Park Yeon-kyo'),
#  Document(metadata={'source': 'movies/3.txt'}, page_content='Title: The Matrix\n   Director: The Wachowskis\n   Release Year: 1999\n   Genre: Action, Sci-Fi\n   Rating: 8.7\n   Cast:\n     - Keanu Reeves as Neo\n     - Laurence Fishburne as Morpheus\n     - Carrie-Anne Moss as Trinity'),
#  Document(metadata={'source': 'movies/1.txt'}, page_content='Title: Inception\n   Director: Christopher Nolan\n   Release Year: 2010\n   Genre: Action, Sci-Fi, Thriller\n   Rating: 8.8\n   Cast:\n     - Leonardo DiCaprio as Dom Cobb\n     - Joseph Gordon-Levitt as Arthur\n     - Elliot Page as Ariadne'),
#  Document(metadata={'source': 'movies/2.txt'}, page_content="Title: The Shawshank Redemption\n   Director: Frank Darabont\n   Release Year: 1994\n   Genre: Drama\n   Rating: 9.3\n   Cast:\n     - Tim Robbins as Andy Dufresne\n     - Morgan Freeman as Ellis Boyd 'Red' Redding")]
```

