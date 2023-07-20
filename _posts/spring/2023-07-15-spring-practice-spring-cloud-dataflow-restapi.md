--- 
layout: single
classes: wide
title: "[Spring 실습] "
header:
  overlay_image: /img/spring-bg.jpg
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Spring
tags:
    - Practice
    - Spring
    - Spring Boot
    - Spring Cloud Data Flow
    - SCDF
toc: true
use_math: true
---  

## Spring Cloud DataFlow REST API
[Spring Cloud DataFlow REST API](https://docs.spring.io/spring-cloud-dataflow/docs/2.10.3/reference/htmlsingle/#api-guide) 
를 사용하면 추가적인 툴필요 없이 `REST API` 만으로 `Spring Cloud DataFlow` 를 조작할 수 있다.  

`REST API` 에 대한 전체 `Endpoints` 는 아래 요청으로 확인해 볼 수 있다.  

```bash
$ curl -X GET http://localhost:9393 | jq
{
  "_links": {
    "dashboard": {
      "href": "http://localhost:9393/dashboard"
    },
    "audit-records": {
      "href": "http://localhost:9393/audit-records"
    },
    "streams/definitions": {
      "href": "http://localhost:9393/streams/definitions"
    },
    "streams/definitions/definition": {
      "href": "http://localhost:9393/streams/definitions/{name}",
      "templated": true
    },
    "streams/validation": {
      "href": "http://localhost:9393/streams/validation/{name}",
      "templated": true
    },
    "runtime/streams": {
      "href": "http://localhost:9393/runtime/streams{?names}",
      "templated": true
    },
    "runtime/streams/{streamNames}": {
      "href": "http://localhost:9393/runtime/streams/{streamNames}",
      "templated": true
    },
    "runtime/apps": {
      "href": "http://localhost:9393/runtime/apps"
    },
    "runtime/apps/{appId}": {
      "href": "http://localhost:9393/runtime/apps/{appId}",
      "templated": true
    },
    
    .. 생략 ..
}
```  

그리고 `REST API` 에서 제공하는 전체 `Resources` 는 [공식 문서](https://docs.spring.io/spring-cloud-dataflow/docs/2.10.3/reference/htmlsingle/#api-guide-resources)
에서 확인 할 수 있다.  

### SCDF 정보 (/about)

`GET {scdf-ip}:{scdf-port}/about` 요청은 `SCDF` 의 버전 및 상세 정보를 확인 할 수 있다.  

```bash
$ curl -X GET http://localhost:9393/about | jq
{
  "featureInfo": {
    "analyticsEnabled": true,
    "streamsEnabled": true,
    "tasksEnabled": true,
    "schedulesEnabled": true,
    "monitoringDashboardType": "NONE"
  },
  "versionInfo": {
    "implementation": {
      "name": "spring-cloud-dataflow-server",
      "version": "2.9.1"
    },
    "core": {
      "name": "Spring Cloud Data Flow Core",
      "version": "2.9.1"
    },
    "dashboard": {
      "name": "Spring Cloud Dataflow UI",
      "version": "3.2.1"
    },
    "shell": {
      "name": "Spring Cloud Data Flow Shell",
      "version": "2.9.1",
      "url": "https://repo1.maven.org/maven2/org/springframework/cloud/spring-cloud-dataflow-shell/2.9.1/spring-cloud-dataflow-shell-2.9.1.jar"
    }
  },
  "securityInfo": {
    "authenticationEnabled": false,
    "authenticated": false,
    "username": null,
    "roles": []
  },
  "runtimeEnvironment": {
    "appDeployer": {
      "deployerImplementationVersion": "2.8.1",
      "deployerName": "Spring Cloud Skipper Server",
      "deployerSpiVersion": "2.8.1",
      "javaVersion": "11.0.13",
      "platformApiVersion": "",
      "platformClientVersion": "",
      "platformHostVersion": "",
      "platformSpecificInfo": {
        "default": "kubernetes"
      },
      "platformType": "Skipper Managed",
      "springBootVersion": "2.5.5",
      "springVersion": "5.3.10"
    },
    "taskLaunchers": [
      {
        "deployerImplementationVersion": "2.7.1",
        "deployerName": "KubernetesTaskLauncher",
        "deployerSpiVersion": "2.7.1",
        "javaVersion": "11.0.13",
        "platformApiVersion": "v1",
        "platformClientVersion": "unknown",
        "platformHostVersion": "unknown",
        "platformSpecificInfo": {
          "namespace": "default",
          "master-url": "https://172.24.0.1:443/"
        },
        "platformType": "Kubernetes",
        "springBootVersion": "2.5.5",
        "springVersion": "5.3.10"
      }
    ]
  },
  "monitoringDashboardInfo": {
    "url": "",
    "refreshInterval": 15,
    "dashboardType": "NONE",
    "source": "default-scdf-source"
  },
  "_links": {
    "self": {
      "href": "http://localhost:9393/about"
    }
  }
}
```  

### 애플리케이션 추가 (/apps/{type}/{name})
`POST {scdf-ip}:{scdf-port}/apps/{type}/{name}` 요청을 통해 새로운 애플리케이션을 추가 할 수 있다.  
테스트를 위해 `Processor` 애플리케이션 중 [uppercase-transformer-kafka](https://hub.docker.com/r/springcloudstream/uppercase-transformer-kafka)
라는 애플리케이션을 추가한다.


```
$ curl -X POST http://localhost:9393/apps/processor/uppercase-transformer-kafka \
-d 'uri=docker:springcloudstream/uppercase-transformer-kafka'
```  
