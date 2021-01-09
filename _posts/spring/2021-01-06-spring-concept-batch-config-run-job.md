--- 
layout: single
classes: wide
title: "[Spring 개념] "
header:
  overlay_image: /img/spring-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Concept
    - Spring
    - Spring Batch
    - Job
    - Step
    - ExecutionContext
toc: true
use_math: true
---  

## Job 설정 및 실행

### Job 설정
`Job` 은 `JobBuilderFatory` 를 이용해서 아래와 같이 간단하게 빈으로 생성할 수 있다. 

```java
@Bean
public Job simpleJob() {
    return this.jobBuilderFactory.get("simpleJob")
            
}


```  

---
## Reference
[Configuring and Running a Job](https://docs.spring.io/spring-batch/docs/4.3.x/reference/html/job.html#configureJob)  