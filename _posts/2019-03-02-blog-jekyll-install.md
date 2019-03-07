--- 
layout: single
classes: wide
title: "윈도우에서 Jekyll 환경 구성하기"
header:
  overlay_image: /img/blog-bg.jpg
excerpt: '윈도우에서 Jekyll 구성하는 법을 알아보자'
author: "window_for_sun"
header-style: text
toc: true
categories :
  - Blog
tags:
    - Blog
    - Jekyll
    - Intro
---  

## Ruby 설치
- [Ruby Downloads](https://rubyinstaller.org/downloads/) 링크로 가서 윈도우용 With DevKit 설치 프로그램을 다운로드 받는다.
- 다운로드한 Installer 를 누르면 Ruby 설치 및 환경변수(Path) 설정이 완료 된다.
![루비 다운로드 홈페이지]({{site.basesurl}}/img/blog-jekyll-install-rubydownload.png)

## Jekyll 설치하기
- 윈도우 검색을 사용해서 **Start Command Prompt with Ruby** 를 실행 시킨다.
![윈도우에서 Start Command Prompt with Ruby 검색하기]({{site.baseurl}}/img/blog-jekyll-install-searchrubyinwindow.png)
- 실행 시킨 콘솔창에 아래 명령어로 필요한 패키지를 다운로드 한다.

```
gem install jekyll
gem install minima
gem install bundler
gem install jekyll-feed
gem install tzinfo-data
```  

## 블로그 연동하기
- 위 설치시에 실행했던 콘솔창에서 로컬로 연동 시키고 싶은 블로그 경로로 이동한다.
- 아래 명령어로 현재 블로그에 필요한 bundle 을 설치 및 업데이트 한다.

```
bundle install // 초기
bundle install // 이후 변경사항이 있을 경우
```  

- 아래 명령어를 통해 Jekyll 을 실행 시킨다.

```
jekyll serve // 안된다면 아래것
bundle exec jekyll serve
```  



---
## Reference
[지킬 한국어 페이지](https://jekyllrb-ko.github.io/)  



