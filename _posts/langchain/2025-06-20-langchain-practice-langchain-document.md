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


### JSONLoader
`JSONLoader` 는 `JSON` 형식 파일을 로드하여 `Document` 객체로 변환하는 로더이다. 
복잡한 구조의 `JSON` 데이터에서 특정 정보를 추출하는데 유용하다. 

주요 특징은 아래와 같다.  

- `JSON` 파일에서 특정 필드나 경로의 데이터를 추출할 수 있다. 
- `jq` 와 유사한 표현식을 사용하여 복잡한 `JSON` 구조를 탐색할 수 있다. 
- 추출된 데이터를 `Document` 객체로 변환한다. 
- 메타 데이터 추출 기능을 제공한다.  


주요 매개변수는 아래와 같다. 

```python
JSONLoader(
    # 로드할 `JSON` 파일 경로
    file_path: str,
    # JSON 데이터를 탐색하기 위한 jq 스타일 표현식, 기본값 .[]
    jq_schema: str = '.[]',
    # 문서 내용으로 사용할 JSON 필드의 키
    content_key: Optional[str] = None,
    # 메타데이터를 추출하기 위한 사용자 정의 변수
    metadata_func: Optional[Callable[[Dict, Dict], Dict]] = None
)
```  

아래와 같은 영화관련 정보가 담긴 `JSON` 파일을 로드해 사용한다.  

<details><summary>JSON 내용</summary>
<div markdown="1">

```json
{
  "movies": [
    {
      "title": "Inception",
      "director": "Christopher Nolan",
      "release_year": 2010,
      "genre": ["Action", "Sci-Fi", "Thriller"],
      "rating": 8.8,
      "cast": [
        {
          "name": "Leonardo DiCaprio",
          "role": "Dom Cobb"
        },
        {
          "name": "Joseph Gordon-Levitt",
          "role": "Arthur"
        },
        {
          "name": "Elliot Page",
          "role": "Ariadne"
        }
      ]
    },
    {
      "title": "The Shawshank Redemption",
      "director": "Frank Darabont",
      "release_year": 1994,
      "genre": ["Drama"],
      "rating": 9.3,
      "cast": [
        {
          "name": "Tim Robbins",
          "role": "Andy Dufresne"
        },
        {
          "name": "Morgan Freeman",
          "role": "Ellis Boyd 'Red' Redding"
        }
      ]
    },
    {
      "title": "The Matrix",
      "director": "The Wachowskis",
      "release_year": 1999,
      "genre": ["Action", "Sci-Fi"],
      "rating": 8.7,
      "cast": [
        {
          "name": "Keanu Reeves",
          "role": "Neo"
        },
        {
          "name": "Laurence Fishburne",
          "role": "Morpheus"
        },
        {
          "name": "Carrie-Anne Moss",
          "role": "Trinity"
        }
      ]
    },
    {
      "title": "Parasite",
      "director": "Bong Joon-ho",
      "release_year": 2019,
      "genre": ["Drama", "Thriller"],
      "rating": 8.5,
      "cast": [
        {
          "name": "Song Kang-ho",
          "role": "Kim Ki-taek"
        },
        {
          "name": "Lee Sun-kyun",
          "role": "Park Dong-ik"
        },
        {
          "name": "Cho Yeo-jeong",
          "role": "Park Yeon-kyo"
        }
      ]
    },
    {
      "title": "Avengers: Endgame",
      "director": "Anthony Russo, Joe Russo",
      "release_year": 2019,
      "genre": ["Action", "Adventure", "Sci-Fi"],
      "rating": 8.4,
      "cast": [
        {
          "name": "Robert Downey Jr.",
          "role": "Tony Stark / Iron Man"
        },
        {
          "name": "Chris Evans",
          "role": "Steve Rogers / Captain America"
        },
        {
          "name": "Scarlett Johansson",
          "role": "Natasha Romanoff / Black Widow"
        }
      ]
    },
    {
      "title": "Spirited Away",
      "director": "Hayao Miyazaki",
      "release_year": 2001,
      "genre": ["Animation", "Adventure", "Fantasy"],
      "rating": 8.6,
      "cast": [
        {
          "name": "Rumi Hiiragi",
          "role": "Chihiro Ogino (voice)"
        },
        {
          "name": "Miyu Irino",
          "role": "Haku (voice)"
        },
        {
          "name": "Mari Natsuki",
          "role": "Yubaba / Zeniba (voice)"
        }
      ]
    },
    {
      "title": "The Godfather",
      "director": "Francis Ford Coppola",
      "release_year": 1972,
      "genre": ["Crime", "Drama"],
      "rating": 9.2,
      "cast": [
        {
          "name": "Marlon Brando",
          "role": "Don Vito Corleone"
        },
        {
          "name": "Al Pacino",
          "role": "Michael Corleone"
        },
        {
          "name": "James Caan",
          "role": "Sonny Corleone"
        }
      ]
    },
    {
      "title": "Interstellar",
      "director": "Christopher Nolan",
      "release_year": 2014,
      "genre": ["Adventure", "Drama", "Sci-Fi"],
      "rating": 8.6,
      "cast": [
        {
          "name": "Matthew McConaughey",
          "role": "Cooper"
        },
        {
          "name": "Anne Hathaway",
          "role": "Brand"
        },
        {
          "name": "Jessica Chastain",
          "role": "Murph"
        }
      ]
    }
  ]
}
```

</div>
</details>  

아래 예제는 `JSON` 내용 중 주요 출연진(`cast`) 정보만을 추출하는 예제이다.  


```python
from langchain_community.document_loaders import JSONLoader

loader = JSONLoader(
    file_path="movies.json",
    jq_schema='.movies[].cast',
    text_content=False
)

docs = loader.load()
# [Document(metadata={'source': '/content/movies.json', 'seq_num': 1}, page_content='[{"name": "Leonardo DiCaprio", "role": "Dom Cobb"}, {"name": "Joseph Gordon-Levitt", "role": "Arthur"}, {"name": "Elliot Page", "role": "Ariadne"}]'),
# Document(metadata={'source': '/content/movies.json', 'seq_num': 2}, page_content='[{"name": "Tim Robbins", "role": "Andy Dufresne"}, {"name": "Morgan Freeman", "role": "Ellis Boyd \'Red\' Redding"}]'),
# Document(metadata={'source': '/content/movies.json', 'seq_num': 3}, page_content='[{"name": "Keanu Reeves", "role": "Neo"}, {"name": "Laurence Fishburne", "role": "Morpheus"}, {"name": "Carrie-Anne Moss", "role": "Trinity"}]'),
# Document(metadata={'source': '/content/movies.json', 'seq_num': 4}, page_content='[{"name": "Song Kang-ho", "role": "Kim Ki-taek"}, {"name": "Lee Sun-kyun", "role": "Park Dong-ik"}, {"name": "Cho Yeo-jeong", "role": "Park Yeon-kyo"}]'),
# Document(metadata={'source': '/content/movies.json', 'seq_num': 5}, page_content='[{"name": "Robert Downey Jr.", "role": "Tony Stark / Iron Man"}, {"name": "Chris Evans", "role": "Steve Rogers / Captain America"}, {"name": "Scarlett Johansson", "role": "Natasha Romanoff / Black Widow"}]'),
# Document(metadata={'source': '/content/movies.json', 'seq_num': 6}, page_content='[{"name": "Rumi Hiiragi", "role": "Chihiro Ogino (voice)"}, {"name": "Miyu Irino", "role": "Haku (voice)"}, {"name": "Mari Natsuki", "role": "Yubaba / Zeniba (voice)"}]'),
# Document(metadata={'source': '/content/movies.json', 'seq_num': 7}, page_content='[{"name": "Marlon Brando", "role": "Don Vito Corleone"}, {"name": "Al Pacino", "role": "Michael Corleone"}, {"name": "James Caan", "role": "Sonny Corleone"}]'),
# Document(metadata={'source': '/content/movies.json', 'seq_num': 8}, page_content='[{"name": "Matthew McConaughey", "role": "Cooper"}, {"name": "Anne Hathaway", "role": "Brand"}, {"name": "Jessica Chastain", "role": "Murph"}]')]
```  

그리고 다음은 영화 제목과 감독을 메타정보로 추가하는 예제이다.  


```python
from langchain_community.document_loaders import JSONLoader

def metadata_func(record: dict, metadata: dict) -> dict:
    metadata["title"] = record.get("title", "")
    metadata["director"] = record.get("director", "")
    return metadata

loader = JSONLoader(
    file_path="movies.json",
    jq_schema='.movies[]',
    text_content=False,
    # jq_schema 의 전체 내용을 로드하고 싶다면 None 으로 설정하면 된다. 
    content_key='cast',
    metadata_func=metadata_func
)

docs = loader.load()
# [Document(metadata={'source': '/content/movies.json', 'seq_num': 1, 'title': 'Inception', 'director': 'Christopher Nolan'}, page_content='[{"name": "Leonardo DiCaprio", "role": "Dom Cobb"}, {"name": "Joseph Gordon-Levitt", "role": "Arthur"}, {"name": "Elliot Page", "role": "Ariadne"}]'),
#  Document(metadata={'source': '/content/movies.json', 'seq_num': 2, 'title': 'The Shawshank Redemption', 'director': 'Frank Darabont'}, page_content='[{"name": "Tim Robbins", "role": "Andy Dufresne"}, {"name": "Morgan Freeman", "role": "Ellis Boyd \'Red\' Redding"}]'),
#  Document(metadata={'source': '/content/movies.json', 'seq_num': 3, 'title': 'The Matrix', 'director': 'The Wachowskis'}, page_content='[{"name": "Keanu Reeves", "role": "Neo"}, {"name": "Laurence Fishburne", "role": "Morpheus"}, {"name": "Carrie-Anne Moss", "role": "Trinity"}]'),
#  Document(metadata={'source': '/content/movies.json', 'seq_num': 4, 'title': 'Parasite', 'director': 'Bong Joon-ho'}, page_content='[{"name": "Song Kang-ho", "role": "Kim Ki-taek"}, {"name": "Lee Sun-kyun", "role": "Park Dong-ik"}, {"name": "Cho Yeo-jeong", "role": "Park Yeon-kyo"}]'),
#  Document(metadata={'source': '/content/movies.json', 'seq_num': 5, 'title': 'Avengers: Endgame', 'director': 'Anthony Russo, Joe Russo'}, page_content='[{"name": "Robert Downey Jr.", "role": "Tony Stark / Iron Man"}, {"name": "Chris Evans", "role": "Steve Rogers / Captain America"}, {"name": "Scarlett Johansson", "role": "Natasha Romanoff / Black Widow"}]'),
#  Document(metadata={'source': '/content/movies.json', 'seq_num': 6, 'title': 'Spirited Away', 'director': 'Hayao Miyazaki'}, page_content='[{"name": "Rumi Hiiragi", "role": "Chihiro Ogino (voice)"}, {"name": "Miyu Irino", "role": "Haku (voice)"}, {"name": "Mari Natsuki", "role": "Yubaba / Zeniba (voice)"}]'),
#  Document(metadata={'source': '/content/movies.json', 'seq_num': 7, 'title': 'The Godfather', 'director': 'Francis Ford Coppola'}, page_content='[{"name": "Marlon Brando", "role": "Don Vito Corleone"}, {"name": "Al Pacino", "role": "Michael Corleone"}, {"name": "James Caan", "role": "Sonny Corleone"}]'),
#  Document(metadata={'source': '/content/movies.json', 'seq_num': 8, 'title': 'Interstellar', 'director': 'Christopher Nolan'}, page_content='[{"name": "Matthew McConaughey", "role": "Cooper"}, {"name": "Anne Hathaway", "role": "Brand"}, {"name": "Jessica Chastain", "role": "Murph"}]')]
```  


`JSONLoader` 는 표준 `JSON` 파일을 처리하는 로더이다. 
만약 각 줄이 독립적인 `JSON` 객체인 `JSONL` 형식 파일 처리가 필요하다면 `JSONLinesLoader` 를 사용하면 된다.  


### CSVLoader
`CSVLoader` 는 `CSV` 형식 파일을 로드해 `Document` 객체로 변환하는 문서 로더이다. 
`CSV` 파일의 각 행(`row`)을 별도의 `Document` 객체로 변환하고, 
헤더와 값을 결합하여 구조화된 텍스트로 변환할 수 있다. 
그리고 특정 열만 선택해 로드하는 것도 가능하다. 
또한 사용자 정의 구분자를 지원해 쉽표 외의 구분자도 사용 가능하다.  

주요 매개변수는 아래와 같다. 

```python
CSVLoader(
    # 로드할 CSV 파일 경로
    file_path: str,
    # 메타데이터의 source 필드에 사용할 열이름
    source_column: Optional[str] = None,
    # csv 모듈의 reader 에 전달할 인자(delimiter, quotechar 등)
    csv_args: Optional[Dict] = None,
    # 파일을 읽을 때 사용할 인코딩, 기본값 None(시스템 기본값 사용)
    encoding: Optional[str] = None,
    # 인코딩을 자동으로 감지할지 여부, 기본값 False
    autodetect_encoding: bool = False,
    # 메타데이터로 추출할 열 목록
    metadata_columns: List[str] = []
)
```  

예제는 아래와 같은 `CSV` 형식으로 작성된 영화관련 정보를 사용한다.  

<details><summary>CSV 내용</summary>
<div markdown="1">

```csv
title,director,release_year,genre,rating,cast_name,cast_role
Inception,Christopher Nolan,2010,"Action|Sci-Fi|Thriller",8.8,Leonardo DiCaprio,Dom Cobb
Inception,Christopher Nolan,2010,"Action|Sci-Fi|Thriller",8.8,Joseph Gordon-Levitt,Arthur
Inception,Christopher Nolan,2010,"Action|Sci-Fi|Thriller",8.8,Elliot Page,Ariadne
The Shawshank Redemption,Frank Darabont,1994,"Drama",9.3,Tim Robbins,Andy Dufresne
The Shawshank Redemption,Frank Darabont,1994,"Drama",9.3,Morgan Freeman,Ellis Boyd 'Red' Redding
The Matrix,The Wachowskis,1999,"Action|Sci-Fi",8.7,Keanu Reeves,Neo
The Matrix,The Wachowskis,1999,"Action|Sci-Fi",8.7,Laurence Fishburne,Morpheus
The Matrix,The Wachowskis,1999,"Action|Sci-Fi",8.7,Carrie-Anne Moss,Trinity
Parasite,Bong Joon-ho,2019,"Drama|Thriller",8.5,Song Kang-ho,Kim Ki-taek
Parasite,Bong Joon-ho,2019,"Drama|Thriller",8.5,Lee Sun-kyun,Park Dong-ik
Parasite,Bong Joon-ho,2019,"Drama|Thriller",8.5,Cho Yeo-jeong,Park Yeon-kyo
Avengers: Endgame,"Anthony Russo, Joe Russo",2019,"Action|Adventure|Sci-Fi",8.4,Robert Downey Jr.,Tony Stark / Iron Man
Avengers: Endgame,"Anthony Russo, Joe Russo",2019,"Action|Adventure|Sci-Fi",8.4,Chris Evans,Steve Rogers / Captain America
Avengers: Endgame,"Anthony Russo, Joe Russo",2019,"Action|Adventure|Sci-Fi",8.4,Scarlett Johansson,Natasha Romanoff / Black Widow
Spirited Away,Hayao Miyazaki,2001,"Animation|Adventure|Fantasy",8.6,Rumi Hiiragi,Chihiro Ogino (voice)
Spirited Away,Hayao Miyazaki,2001,"Animation|Adventure|Fantasy",8.6,Miyu Irino,Haku (voice)
Spirited Away,Hayao Miyazaki,2001,"Animation|Adventure|Fantasy",8.6,Mari Natsuki,Yubaba / Zeniba (voice)
The Godfather,Francis Ford Coppola,1972,"Crime|Drama",9.2,Marlon Brando,Don Vito Corleone
The Godfather,Francis Ford Coppola,1972,"Crime|Drama",9.2,Al Pacino,Michael Corleone
The Godfather,Francis Ford Coppola,1972,"Crime|Drama",9.2,James Caan,Sonny Corleone
Interstellar,Christopher Nolan,2014,"Adventure|Drama|Sci-Fi",8.6,Matthew McConaughey,Cooper
Interstellar,Christopher Nolan,2014,"Adventure|Drama|Sci-Fi",8.6,Anne Hathaway,Brand
Interstellar,Christopher Nolan,2014,"Adventure|Drama|Sci-Fi",8.6,Jessica Chastain,Murph
```  


</div>
</details>  


```python
from langchain_community.document_loaders.csv_loader import CSVLoader

loader = CSVLoader(
    file_path="./movies.csv",
    csv_args={
        "delimiter" : ",",
        "quotechar" : '"',
        "fieldnames" : ["title", "director", "cast_name", "cast_role"]
    },
    source_column="title"
)

docs = loader.load()
# [Document(metadata={'source': 'title', 'row': 0}, page_content='title: title\ndirector: director\ncast_name: release_year\ncast_role: genre\nNone: rating,cast_name,cast_role'),
# Document(metadata={'source': 'Inception', 'row': 1}, page_content='title: Inception\ndirector: Christopher Nolan\ncast_name: 2010\ncast_role: Action|Sci-Fi|Thriller\nNone: 8.8,Leonardo DiCaprio,Dom Cobb'),
# Document(metadata={'source': 'Inception', 'row': 2}, page_content='title: Inception\ndirector: Christopher Nolan\ncast_name: 2010\ncast_role: Action|Sci-Fi|Thriller\nNone: 8.8,Joseph Gordon-Levitt,Arthur'),
# Document(metadata={'source': 'Inception', 'row': 3}, page_content='title: Inception\ndirector: Christopher Nolan\ncast_name: 2010\ncast_role: Action|Sci-Fi|Thriller\nNone: 8.8,Elliot Page,Ariadne'),
# Document(metadata={'source': 'The Shawshank Redemption', 'row': 4}, page_content='title: The Shawshank Redemption\ndirector: Frank Darabont\ncast_name: 1994\ncast_role: Drama\nNone: 9.3,Tim Robbins,Andy Dufresne'),
# Document(metadata={'source': 'The Shawshank Redemption', 'row': 5}, page_content="title: The Shawshank Redemption\ndirector: Frank Darabont\ncast_name: 1994\ncast_role: Drama\nNone: 9.3,Morgan Freeman,Ellis Boyd 'Red' Redding"),
# Document(metadata={'source': 'The Matrix', 'row': 6}, page_content='title: The Matrix\ndirector: The Wachowskis\ncast_name: 1999\ncast_role: Action|Sci-Fi\nNone: 8.7,Keanu Reeves,Neo'),
# ...
```  

그리고 `UnstrcuturedCSVLoader` 를 사용하면 `CSV` 파일의 내용을 [Unstructured](https://github.com/Unstructured-IO/unstructured)
라이브러리를 기반으로 `Document` 객체를 생성할 수 있다. 
`CSVLoader` 를 사용하는 것보다 좀 더 강력한 `Unstructured` 라이브러리르 사용하기 때문에 자동으로 구조를 인식한다던가 풍부한 메타정보를 제공 및 유연한 처리에 장점이 있다. 

```python
from langchain_community.document_loaders.csv_loader import UnstructuredCSVLoader

loader = UnstructuredCSVLoader(
    file_path='./movies.csv',
    csv_args={
        'delimiter': ',',
        'quotechar': '"',
        'fieldnames': ['title', 'director', 'cast_name', 'cast_role']
    },
    mode='elements'
)

docs = loader.load()

docs[0].metadata.keys()
# dict_keys(['source', 'file_directory', 'filename', 'last_modified', 'text_as_html', 'languages', 'filetype', 'category', 'element_id'])
docs[0].metadata['text_as_html']
# <table><tr><td>title</td><td>director</td><td>release_year</td><td>genre</td><td>rating</td><td>cast_name</td><td>cast_role</td></tr><tr><td>Inception</td><td>Christopher Nolan</td><td>2010</td><td>Action|Sci-Fi|Thriller</td><td>8.8</td><td>Leonardo DiCaprio</td><td>Dom Cobb</td></tr><tr><td>Inception</td><td>Christopher Nolan</td><td>2010</td><td>Action|Sci-Fi|Thriller</td><td>8.8</td><td>Joseph Gordon-Levitt</td><td>Arthur</td></tr><tr><td>Inception</td><td>Christopher Nolan</td><td>2010</td><td>Action|Sci-Fi|Thriller</td><td>8.8</td><td>Elliot Page</td><td>Ariadne</td></tr><tr><td>The Shawshank Redemption</td><td>Frank Darabont</td><td>1994</td><td>Drama</td><td>9.3</td><td>Tim Robbins</td><td>Andy Dufresne</td></tr><tr><td>The Shawshank Redemption</td><td>Frank Darabont</td><td>1994</td><td>Drama</td><td>9.3</td><td>Morgan Freeman</td><td>Ellis Boyd 'Red' Redding</td></tr><tr><td>The Matrix</td><td>The Wachowskis</td><td>1999</td><td>Action|Sci-Fi</td><td>8.7</td><td>Keanu Reeves</td><td>Neo</td></tr><tr><td>The Matrix</td><td>The Wachowskis</td><td>1999</td><td>Action|Sci-Fi</td><td>8.7</td><td>Laurence Fishburne</td><td>Morpheus</td></tr><tr><td>The Matrix</td><td>The Wachowskis</td><td>1999</td><td>Action|Sci-Fi</td><td>8.7</td><td>Carrie-Anne Moss</td><td>Trinity</td></tr><tr><td>Parasite</td><td>Bong Joon-ho</td><td>2019</td><td>Drama|Thriller</td><td>8.5</td><td>Song Kang-ho</td><td>Kim Ki-taek</td></tr><tr><td>Parasite</td><td>Bong Joon-ho</td><td>2019</td><td>Drama|Thriller</td><td>8.5</td><td>Lee Sun-kyun</td><td>Park Dong-ik</td></tr><tr><td>Parasite</td><td>Bong Joon-ho</td><td>2019</td><td>Drama|Thriller</td><td>8.5</td><td>Cho Yeo-jeong</td><td>Park Yeon-kyo</td></tr><tr><td>Avengers: Endgame</td><td>Anthony Russo, Joe Russo</td><td>2019</td><td>Action|Adventure|Sci-Fi</td><td>8.4</td><td>Robert Downey Jr.</td><td>Tony Stark / Iron Man</td></tr><tr><td>Avengers: Endgame</td><td>Anthony Russo, Joe Russo</td><td>2019</td><td>Action|Adventure|Sci-Fi</td><td>8.4</td><td>Chris Evans</td><td>Steve Rogers / Captain America</td></tr><tr><td>Avengers: Endgame</td><td>Anthony Russo, Joe Russo</td><td>2019</td><td>Action|Adventure|Sci-Fi</td><td>8.4</td><td>Scarlett Johansson</td><td>Natasha Romanoff / Black Widow</td></tr><tr><td>Spirited Away</td><td>Hayao Miyazaki</td><td>2001</td><td>Animation|Adventure|Fantasy</td><td>8.6</td><td>Rumi Hiiragi</td><td>Chihiro Ogino (voice)</td></tr><tr><td>Spirited Away</td><td>Hayao Miyazaki</td><td>2001</td><td>Animation|Adventure|Fantasy</td><td>8.6</td><td>Miyu Irino</td><td>Haku (voice)</td></tr><tr><td>Spirited Away</td><td>Hayao Miyazaki</td><td>2001</td><td>Animation|Adventure|Fantasy</td><td>8.6</td><td>Mari Natsuki</td><td>Yubaba / Zeniba (voice)</td></tr><tr><td>The Godfather</td><td>Francis Ford Coppola</td><td>1972</td><td>Crime|Drama</td><td>9.2</td><td>Marlon Brando</td><td>Don Vito Corleone</td></tr><tr><td>The Godfather</td><td>Francis Ford Coppola</td><td>1972</td><td>Crime|Drama</td><td>9.2</td><td>Al Pacino</td><td>Michael Corleone</td></tr><tr><td>The Godfather</td><td>Francis Ford Coppola</td><td>1972</td><td>Crime|Drama</td><td>9.2</td><td>James Caan</td><td>Sonny Corleone</td></tr><tr><td>Interstellar</td><td>Christopher Nolan</td><td>2014</td><td>Adventure|Drama|Sci-Fi</td><td>8.6</td><td>Matthew McConaughey</td><td>Cooper</td></tr><tr><td>Interstellar</td><td>Christopher Nolan</td><td>2014</td><td>Adventure|Drama|Sci-Fi</td><td>8.6</td><td>Anne Hathaway</td><td>Brand</td></tr><tr><td>Interstellar</td><td>Christopher Nolan</td><td>2014</td><td>Adventure|Drama|Sci-Fi</td><td>8.6</td><td>Jessica Chastain</td><td>Murph</td></tr></table>
```  


### DataFrameLoader
`DataFrameLoader` 는 `Pandas DataFrame` 을 `Document` 객체로 변환하는 역할을 한다. 
파일에서 데이터를 읽어오는 대신 이미 메모리에 로드된 `DataFrame` 객체를 직접 처리하 수 있어 데이터 분석과 같은 워크플로우 통합에 유용하다.  

주요한 특징으로는 아래와 같다. 

- 열 이름과 값을 결합하여 구조화된 텍스트를 생성한다. 
- 특정 열만 선택적으로 `Document` 내용에 포함할 수 있다. 
- 메타데이터 열을 지정하여 `Document` 객체에 메타데이터로 저장할 수 있다. 
- 페이지 내용 형식을 사용자가 지정할 수 있다.

다음은 `CSVLoader` 에서 사용했던 영화 정보를 `Pandas` 를 사용해 `DataFrame` 으로 변환한 후
`DataFrameLoader` 를 사용해 `Document` 객체로 변환하는 예제이다.  

```python
import pandas as pd
from langchain_community.document_loaders import DataFrameLoader

df = pd.read_csv('./movies.csv')

loader = DataFrameLoader(
    data_frame=df,
    page_content_column='cast_name'
)

docs = loader.load()
# [Document(metadata={'title': 'Inception', 'director': 'Christopher Nolan', 'release_year': 2010, 'genre': 'Action|Sci-Fi|Thriller', 'rating': 8.8, 'cast_role': 'Dom Cobb'}, page_content='Leonardo DiCaprio'),
#  Document(metadata={'title': 'Inception', 'director': 'Christopher Nolan', 'release_year': 2010, 'genre': 'Action|Sci-Fi|Thriller', 'rating': 8.8, 'cast_role': 'Arthur'}, page_content='Joseph Gordon-Levitt'),
#  Document(metadata={'title': 'Inception', 'director': 'Christopher Nolan', 'release_year': 2010, 'genre': 'Action|Sci-Fi|Thriller', 'rating': 8.8, 'cast_role': 'Ariadne'}, page_content='Elliot Page'),
#  Document(metadata={'title': 'The Shawshank Redemption', 'director': 'Frank Darabont', 'release_year': 1994, 'genre': 'Drama', 'rating': 9.3, 'cast_role': 'Andy Dufresne'}, page_content='Tim Robbins'),
#  Document(metadata={'title': 'The Shawshank Redemption', 'director': 'Frank Darabont', 'release_year': 1994, 'genre': 'Drama', 'rating': 9.3, 'cast_role': "Ellis Boyd 'Red' Redding"}, page_content='Morgan Freeman'),
#  Document(metadata={'title': 'The Matrix', 'director': 'The Wachowskis', 'release_year': 1999, 'genre': 'Action|Sci-Fi', 'rating': 8.7, 'cast_role': 'Neo'}, page_content='Keanu Reeves'),
#  ...
```  


### WebBaseLoader
`WebBaseLoader` 는 웹에서 콘텐츠를 가져와 `Document` 객체로 변환하는 역할을 한다. 
웹 페이지의 `HTML` 내용을 추출하고 처리하여 텍스트 형태로 변환함으로써 `LLM` 이 웹 정보를 활용할 수 있게 한다.  

주요 특징은 아래와 같다.

- 하나 또는 여러 개의 `URL` 에서 콘텐츠를 로드한다. 
- `HTTP` 요청을 통해 웹 페이지 내용을 가져온다. 
- `HTML` 태그를 제거하고 텍스트 내용만 추출한다. 
- 각 웹 페이지를 별도의 `Document` 객체로 변환한다. 
- 오쳥 헤더 및 기타 `HTTP` 욥선을 사용자 정의할 수 있다.  
- `BeautifulSoup` 를 사용해 웹 페이지를 파싱하고 처리한다.

주요 매개변수는 아래와 같다.  

```python
WebBaseLoader(
    # 로드할 웹 URL 또는 URL 목록
    web_paths: Union[str, List[str]],
    # 요청에 사용할 헤더
    header_template: Optional[Dict[str, str]] = None,
    # SSL 인증서 검증 여부
    verify_ssl: bool = True,
    # 초당 최대 요청 수 제한
    requests_per_second: Optional[int] = None,
    # requests 라이브러리에 전달할 추가 인자
    requests_kwargs: Optional[Dict[str, Any]] = None,
    # BeautifulSoup에 전달할 추가 인자
    bs_kwargs: Optional[Dict[str, Any]] = None,
    # HTTP 에러 발생 시 예외를 발생시킬지 여부
    raise_for_status: bool = True,
    # 페이지 인코딩 방식 지정
    encoding: Optional[str] = None
)
```  

`WebBaseLoader` 는 기본적으로 `Javascript` 로 렌더링되는 콘텐츠는 가져올 수 없다. 
이런 경우 `SeleniumURLLoader` 를 고려해야 한다.  

다음은 뉴스기사를 가져와 `Document` 객체로 변환하는 예제이다.  

```python
from ast import parse
import bs4
from langchain_community.document_loaders import WebBaseLoader

path_list = [
    "https://n.news.naver.com/mnews/article/021/0002701072?sid=104",
    "https://n.news.naver.com/mnews/article/015/0005115365?sid=105",
    "https://n.news.naver.com/mnews/article/092/0002369364?sid=105",
    "https://n.news.naver.com/mnews/article/421/0008170130?sid=105",
    "https://n.news.naver.com/mnews/article/001/0015304353?sid=104"
]

loader = WebBaseLoader(
    web_paths=path_list,
    bs_kwargs=dict(
        parse_only=bs4.SoupStrainer(
            'div',
            attrs={'class': ["newsct_article _article_body", "media_end_head_title"]}
        )
    ),
    header_template={
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/102.0.0.0 Safari/537.36",
    },
    # ssl 인증 우회가 필요한 경우
    # requests_kwargs={'verify':False}
)


docs = loader.load()

len(docs)
# 5
docs[0]
# Document(metadata={'source': 'https://n.news.naver.com/mnews/article/021/0002701072?sid=104'}, page_content='\n전세계 ‘지브리 프샤‘ 열풍…오픈AI, 연 매출 20조 가능할까\n\n\n\t\t\tGPT-4o 기반 이미지 생성 기술 바탕 ‘지브리 프샤’ 돌풍1분기 말 오픈AI 챗GPT 유료 가입자 전 분기 대비 30% 증가올해 목표 127억 달러 달성 가능 분석 제기돼\n\n\n\n다운로드 샘 올트만 오픈AI  CEO의 소셜미디어 X 프로필 이미지.                                              X 제공  1분기 말 유료 구독자 수가 전 분기 대비 30% 가까이 증가하는 등 오픈AI 챗GPT 유료 가입자 수가 빠르게 증가하고 있다. 지난달 말 출시한 이미지 생성 기술이 온라인 상에서 입소문을 타며 이용자 유입에 견인차 역할을 했다는 분석이 나온다. 오픈AI는 올해 매출 목표를 전년 대비 3배 증가한 127억 달러(약 18조3000억 원)를 정한 바 있는데 이를 달성할 수 있을지 주목된다.4일(현지 시간) 디 인포메이션 등에 따르면 오픈AI는 지난 1분기에 12억4500만 달러의 수익을 거둔 것으로 추정된다. 유료 멤버십 구독료, API 사용 수익 등을 종합한 수치로 3개월 만에 분기 수익을 약 30% 끌어올린 셈이다. 오픈AI의 챗GPT 주간이용자수(WAU)와 유료 구독자 수도 지난달 말 기준 각각 5억명, 2000만명을 돌파했다. 지난해 말 대비 WAU는 1억5000만 명, 유료 구독자 수는 450만 명 늘었다. 지난해 12월 출시된 동영상 생성 인공지능(AI) ‘소라(Sora)’와 챗GPT 고급 음성 모드(AVM) 실시간 영상 이해 기능 등이 이용자 증가에 일부 영향을 줬지만 업계에선 지난달 25일 공개한 GPT-4o 기반 이미지 생성 기술이 가장 큰 성장 동력이었다는 평가가 나온다.GPT-4o 기반 이미지 생성 기술은 일본 애니메이션 제작사 ‘스튜디오 지브리’ 화풍으로 이미지를 만들어줄 수 있다는 점에서 대중적 관심을 끌었다. 특정 사진을 게재한 뒤 "지브리 화풍으로 그려줘"라고 요청하면 마치 지브리 애니메이션에 나올 법한 그림을 제공하는데 이용자 호응이 높은 편이다. 이용자들은 챗GPT로 만든 지브리 화풍 그림을 카카오톡, 인스타그램, 엑스 등의 프로필 사진으로 설정했다. 국내외 커뮤니티에는 후기와 생성 사례가 빠르게 퍼지고 있다.샘 올트먼 오픈AI 최고경영자(CEO)도 밈 확산에 동참했다. 그도 자신의 얼굴을 지브리 화풍을 모사한 그림을 엑스 프로필 사진으로 설정했다. 이어 지브리 스타일은 아니지만 아들과 함께 있는 사진을 AI로 제작한 이미지, 인도 크리켓 선수로 가장한 애니메이션 캐릭터 등 다양한 이미지를 선보이면서 챗GPT 사용을 독려했다.그는 지난달 31일 엑스를 통해 "챗GPT를 출시했을 때 100만 이용자를 확보하는 데 5일이 걸렸지만 지금은 단 한 시간 만에 이용자 100만명이 추가됐다"며 이미지 생성 기술 파급력을 강조했다. 브래드 라이트캡 오픈AI 최고운영책임자(COO)도 3일 엑스를 통해 "(출시 후 1주일 동안) 1억3000만명 이상의 사용자가 7억 장 이상의 이미지를 생성했다"며 흥행 성과를 전했다.\n\n')
```  

추가로 `WebBaseLoader` 외 웹 콘텐츠 처리를 위한 아래와 같은 로더들이 있다. 

- `SeleniumURLLoader` : `Selenium` 을 사용해 `Javascript` 렌더링이 필요한 페이지에 사용할 수 있는 로더이다.
- `PlaywrightURLLoader` : `Playwright` 를 사용한 로더이다. 
- `RecursiveUrlLoader` : `WebBaseLoader` 를 사용해 웹 페이지를 재귀적으로 탐색하는 로더이다.
- `SitemapLoader` : 웹사이트의 사이트맵을 활용한 로드이다. 
- `ArxivLoader` : `arxiv` 에서 논문을 로드하는 로더이다.  


### SQLDatabaseLoader
`SQLDatabaseLoader` 는 다양한 `SQL` 데이터베이스에서 데이터를 쿼리하여 `Document` 객체로 변환하는 역할을 한다. 
`MySQL`, `PostgreSQL`, `SQLite` 등 다양한 데이터베이스를 지원하며, 
여러 종류를 단일 인터페이스로 처리할 수 있는 범용적인 로더이다. 

주요 특징은 아래와 같다.

- 다양한 `SQL` 데이터베이스 시스텀에 연결할 수 있다. 
- `SQL` 쿼리를 사용하여 데이터베이스에서 데이터를 추출할 수 있다. 
- 쿼리 결과의 각 행을 별도의 `Document` 객체로 변환한다 
- `SQLAlchemy` 를 사용하여 데이터베이스와 상호작용한다.
- 특정 열을 문서 내용으로, 다른 열은 메타데이터로 처리할 수 있다. 


```python
import sqlite3
from langchain.utilities import SQLDatabase
from langchain_community.document_loaders import SQLDatabaseLoader

conn = sqlite3.connect('movies.db')
cursor = conn.cursor()

# 테이블 생성
cursor.execute('''
CREATE TABLE IF NOT EXISTS movies (
    id INTEGER PRIMARY KEY,
    title TEXT,
    director TEXT,
    release_year INTEGER,
    genre TEXT,
    rating REAL
)
''')

# 데이터 삽입
movies_data = [
    (1, 'Inception', 'Christopher Nolan', 2010, 'Action|Sci-Fi|Thriller', 8.8),
    (2, 'The Shawshank Redemption', 'Frank Darabont', 1994, 'Drama', 9.3),
    (3, 'The Matrix', 'The Wachowskis', 1999, 'Action|Sci-Fi', 8.7),
    (4, 'Parasite', 'Bong Joon-ho', 2019, 'Drama|Thriller', 8.5),
    (5, 'Avengers: Endgame', 'Anthony Russo, Joe Russo', 2019, 'Action|Adventure|Sci-Fi', 8.4)
]


cursor.executemany('INSERT OR REPLACE INTO movies VALUES (?, ?, ?, ?, ?, ?)', movies_data)


conn.commit()
conn.close()

# SQLDatabase 객체 생성
db = SQLDatabase.from_uri("sqlite:///movies.db")

# 기본 쿼리를 사용한 SQLDatabaseLoader
query = "SELECT title, director, release_year, genre, rating FROM movies"

def page_content(record: dict) -> str:
    return f"타이틀:{record.get('title','')},장르:{record.get('genre','')}"

def metadata(record: dict) -> dict:
  metadata = {}
  metadata["title"] = record.get("title", "")
  metadata["director"] = record.get("director", "")
  return metadata

loader = SQLDatabaseLoader(
    db=db,
    query=query,
    page_content_mapper=page_content,
    metadata_mapper=metadata 
)

docs = loader.load()
# [Document(metadata={'title': 'Inception', 'director': 'Christopher Nolan'}, page_content='타이틀:Inception,장르:Action|Sci-Fi|Thriller'),
# Document(metadata={'title': 'The Shawshank Redemption', 'director': 'Frank Darabont'}, page_content='타이틀:The Shawshank Redemption,장르:Drama'),
# Document(metadata={'title': 'The Matrix', 'director': 'The Wachowskis'}, page_content='타이틀:The Matrix,장르:Action|Sci-Fi'),
# Document(metadata={'title': 'Parasite', 'director': 'Bong Joon-ho'}, page_content='타이틀:Parasite,장르:Drama|Thriller'),
# Document(metadata={'title': 'Avengers: Endgame', 'director': 'Anthony Russo, Joe Russo'}, page_content='타이틀:Avengers: Endgame,장르:Action|Adventure|Sci-Fi')]
```


---  
## Reference
[Document](https://python.langchain.com/api_reference/core/documents/langchain_core.documents.base.Document.html)  
[Document](https://wikidocs.net/253706)  
[Document Loader](https://python.langchain.com/docs/integrations/document_loaders/)  



