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

### 애플리케이션 리스트 (/apps)
`GET {scdf-ip}:{scdf-port}/apps` 요청으로 등록된 애플리케이션을 조회할 수 있다. 

```bash
$ curl -X GET http://localhost:9393/apps | jq
{
  "_embedded": {
    "appRegistrationResourceList": [
      {
        "name": "jms",
        "type": "source",
        "uri": "docker:springcloudstream/jms-source-kafka:3.2.1",
        "version": "3.2.1",
        "defaultVersion": true,
        "versions": null,
        "label": null,
        "_links": {
          "self": {
            "href": "http://localhost:9393/apps/source/jms/3.2.1"
          }
        }
      },
      {
        "name": "jdbc",
        "type": "source",
        "uri": "docker:springcloudstream/jdbc-source-kafka:3.2.1",
        "version": "3.2.1",
        "defaultVersion": true,
        "versions": null,
        "label": null,
        "_links": {
          "self": {
            "href": "http://localhost:9393/apps/source/jdbc/3.2.1"
          }
        }
      },
      
      .. 생략 .. 
    ]
  },
  "_links": {
    "first": {
      "href": "http://localhost:9393/apps?page=0&size=20"
    },
    "self": {
      "href": "http://localhost:9393/apps?page=0&size=20"
    },
    "next": {
      "href": "http://localhost:9393/apps?page=1&size=20"
    },
    "last": {
      "href": "http://localhost:9393/apps?page=4&size=20"
    }
  },
  "page": {
    "size": 20,
    "totalElements": 92,
    "totalPages": 5,
    "number": 0
  }
}
```  

애플리케이션 조회는 아래 요청 파라미터를 통해 조건을 추가할 수 있다.  

파라미터|설명
---|---
search|검색 이름
type|애플리케이션 타입(app, source, processor, sink, task)
defaultVersion|기본 버전만 보일지 boolean 값
page|0 부터 조회할 페이지 값
sort|정렬 기준 필드
size|조회할 결과 개수

앞서 추가한 `uppercase-transformer-kafka` 만 조회하는 조건으로 요청을 수행하면 아래와 같다.  

```bash
$ curl -X GET http://localhost:9393/apps\?search\=uppercase\&type\=processor | jq
{
  "_embedded": {
    "appRegistrationResourceList": [
      {
        "name": "uppercase-transformer-kafka",
        "type": "processor",
        "uri": "docker:springcloudstream/uppercase-transformer-kafka",
        "version": "latest",
        "defaultVersion": true,
        "versions": null,
        "label": null,
        "_links": {
          "self": {
            "href": "http://localhost:9393/apps/processor/uppercase-transformer-kafka/latest"
          }
        }
      }
    ]
  },
  "_links": {
    "self": {
      "href": "http://localhost:9393/apps?search=uppercase&type=processor&page=0&size=20"
    }
  },
  "page": {
    "size": 20,
    "totalElements": 1,
    "totalPages": 1,
    "number": 0
  }
}
```  

### 애플리케이션 정보 (/apps/{type}/{name})
`GET /apps/{type}/{name}` 요청은 등록돼 있는 애플리케이션의 상세 정보를 확인 할 수 있다. 
`transform` 애플렠이션의 상세 정보를 조회하면 아래와 같다.  


```bash
$ curl -X GET http://localhost:9033/apps/processor/transform | jq        
{
  "name": "transform",
  "type": "processor",
  "uri": "docker:springcloudstream/transform-processor-kafka:3.2.1",
  "version": "3.2.1",
  "defaultVersion": true,
  "versions": null,
  "label": null,
  "options": [
    {
      "id": "spel.function.expression",
      "name": "expression",
      "type": "java.lang.String",
      "description": "A SpEL expression to apply.",
      "shortDescription": "A SpEL expression to apply.",
      "defaultValue": null,
      "hints": {
        "keyHints": [],
        "keyProviders": [],
        "valueHints": [],
        "valueProviders": []
      },
      "deprecation": null,
      "deprecated": false
    }
  ],
  "shortDescription": null,
  "inboundPortNames": [],
  "outboundPortNames": [],
  "optionGroups": {}
}
```  


### 애플리케이션 삭제 (/apps/{type}/{name}/{version})
`DELETE /apps/{type}/{name}/{version}` 요청은 등록 돼 있는 애플리케이션을 삭제 할 수 있다. 
앞서 추가한 `uppercase-transformer-kafka` 를 삭제하면 아래와 같다.  

```bash
$ curl -X DELETE http://scdf-server.weather.svc.cd1.io.navercorp.com:8088/apps/processor/uppercase-transformer-kafka | jq

.. 조회를 통해 삭제 확인 ..
$ curl -X GET http://scdf-server.weather.svc.cd1.io.navercorp.com:8088/apps\?search\=uppercase\&type\=processor | jq
{
  "_links": {
    "self": {
      "href": "http://scdf-server.weather.svc.cd1.io.navercorp.com:8088/apps?search=uppercase&type=processor&page=0&size=20"
    }
  },
  "page": {
    "size": 20,
    "totalElements": 0,
    "totalPages": 0,
    "number": 0
  }
}
```  

### 스트림 생성 (/streams/definitions)
`POST /streams/definition?deploy=false` 요청을 사용하면 새로운 스트림을 생성 할 수 있다.  

테스트로 생성해볼 스트림은
[Transformer, Router, Tap]({{site.baseurl}}{% link _posts/spring/2023-07-03-spring-practice-spring-scdf-spring-stream-application-transform-router.md %})
와 동일한 스트림을 생성한다.

```bash
$ curl -X POST http://scdf-server.weather.svc.cd1.io.navercorp.com:8088/streams/definitions \
-d 'definition=time | transform | router&name=rest-time-transform-router' | jq
{
  "name": "rest-time-transform-router",
  "dslText": "time | transform | router",
  "originalDslText": "time | transform | router",
  "status": "undeployed",
  "description": "",
  "statusDescription": "The app or group is known to the system, but is not currently deployed",
  "_links": {
    "self": {
      "href": "http://scdf-server.weather.svc.cd1.io.navercorp.com:8088/streams/definitions/rest-time-transform-router"
    }
  }
}

$ curl -X POST http://scdf-server.weather.svc.cd1.io.navercorp.com:8088/streams/definitions \
-d 'definition=:test-router-even > log&name=rest-test-router-even-log' | jq
{
  "name": "rest-test-router-even-log",
  "dslText": ":test-router-even > log",
  "originalDslText": ":test-router-even > log",
  "status": "undeployed",
  "description": "",
  "statusDescription": "The app or group is known to the system, but is not currently deployed",
  "_links": {
    "self": {
      "href": "http://scdf-server.weather.svc.cd1.io.navercorp.com:8088/streams/definitions/rest-test-router-even-log"
    }
  }
}

$ curl -X POST http://scdf-server.weather.svc.cd1.io.navercorp.com:8088/streams/definitions \
-d 'definition=:test-router-odd > log&name=rest-test-router-odd-log' | jq
{
  "name": "rest-test-router-odd-log",
  "dslText": ":test-router-odd > log",
  "originalDslText": ":test-router-odd > log",
  "status": "undeployed",
  "description": "",
  "statusDescription": "The app or group is known to the system, but is not currently deployed",
  "_links": {
    "self": {
      "href": "http://scdf-server.weather.svc.cd1.io.navercorp.com:8088/streams/definitions/rest-test-router-odd-log"
    }
  }
}
```  

### 스트림 리스트 조회 (/streams/definitions)
`GET /streams/definitions` 요청은 정의된 스트림을 검색, 정렬 등을 수행해서 조회할 수 있는데, 
사용 가능한 파라미터는 아래와 같다.  

파라미터|설명
---|---
page|조회 할 페이지
search|조회 할 스트림 이름
sort|정렬 기준 필드 및 방식
size|조회할 크기

`rest` 문자열이 있는 스트림을 조회하면 아래와 같다.  

```bash
$ curl -X GET http://scdf-server.weather.svc.cd1.io.navercorp.com:8088/streams/definitions\?page\=0\&sort\=name,ASC\&search\=rest\&size\=50 | jq

  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  4541    0  4541    0     0   105k      0 --:--:-- --:--:-- --:--:--  119k
{
  "_embedded": {
    "streamDefinitionResourceList": [
      {
        "name": "rest-test-router-even-log",
        "dslText": ":test-router-even > log",
        "originalDslText": ":test-router-even > log",
        "status": "undeployed",
        "description": "",
        "statusDescription": "The app or group is known to the system, but is not currently deployed",
        "_links": {
          "self": {
            "href": "http://scdf-server.weather.svc.cd1.io.navercorp.com:8088/streams/definitions/rest-test-router-even-log"
          }
        }
      },
      {
        "name": "rest-test-router-odd-log",
        "dslText": ":test-router-odd > log",
        "originalDslText": ":test-router-odd > log",
        "status": "undeployed",
        "description": "",
        "statusDescription": "The app or group is known to the system, but is not currently deployed",
        "_links": {
          "self": {
            "href": "http://scdf-server.weather.svc.cd1.io.navercorp.com:8088/streams/definitions/rest-test-router-odd-log"
          }
        }
      },
      {
        "name": "rest-time-transform-router",
        "dslText": "time | transform | router",
        "originalDslText": "time | transform | router",
        "status": "undeployed",
        "description": "",
        "statusDescription": "The app or group is known to the system, but is not currently deployed",
        "_links": {
          "self": {
            "href": "http://scdf-server.weather.svc.cd1.io.navercorp.com:8088/streams/definitions/rest-time-transform-router"
          }
        }
      }
    ]
  },
  "_links": {
    "self": {
      "href": "http://scdf-server.weather.svc.cd1.io.navercorp.com:8088/streams/definitions?search=rest&page=0&size=50&sort=name,asc"
    }
  },
  "page": {
    "size": 50,
    "totalElements": 3,
    "totalPages": 1,
    "number": 0
  }
}

```  

### 스트림 배포 (/streams/deployments/{name})
`POST /streams/deployments/{name}` 요청을 통해서는 정의된 스트림을 실제 환경에 배포 할 수 있다.  


배포할 스트림별 프로퍼티 내용은 아래와 같다.

```properties
## rest-test-router-even-log 
deployer.*.cpu=100m
deployer.*.kubernetes.limits.cpu=2000m
deployer.*.memory=500Mi
deployer.*.kubernetes.limits.memory=1000Mi
app.time.spring.integration.poller.fixed-rate=1000
app.time.date-format=yyyy-MM-dd HH:mm:ss
app.transform.spel.function.expression=payload.substring(payload.length() - 1)
app.router.expression=new Integer(payload) % 2 == 0 ? 'test-router-even' : 'test-router-odd'

## rest-test-router-odd-log  
deployer.*.cpu=100m
deployer.*.kubernetes.limits.cpu=2000m
deployer.*.kubernetes.limits.memory=2000Mi
deployer.*.memory=1000Mi
app.log.name=even-log
spring.cloud.dataflow.skipper.platformName=default

## rest-time-transform-router
deployer.*.cpu=100m
deployer.*.kubernetes.limits.cpu=2000m
deployer.*.kubernetes.limits.memory=2000Mi
deployer.*.memory=1000Mi
app.log.name=odd-log
spring.cloud.dataflow.skipper.platformName=default

```  


```bash
$ curl -X POST http://scdf-server.weather.svc.cd1.io.navercorp.com:8088/streams/deployments/rest-time-transform-router \
-H "Content-Type: application/json" \
--data '{
"deployer.*.cpu":"100m",
"deployer.*.kubernetes.limits.cpu":"2000m",
"deployer.*.memory":"500Mi",
"deployer.*.kubernetes.limits.memory":"1000Mi",
"app.time.spring.integration.poller.fixed-rate":1000,
"app.time.date-format":"yyyy-MM-dd HH:mm:ss",
"app.transform.spel.function.expression":"payload.substring(payload.length() - 1)",
"app.router.expression":"new Integer(payload) % 2 == 0 ? ''test-router-even'' : ''test-router-odd''"
}' | jq

$ curl -X POST http://scdf-server.weather.svc.cd1.io.navercorp.com:8088/streams/deployments/rest-test-router-even-log \
-H "Content-Type: application/json" \
--data '{
"deployer.*.cpu":"100m",
"deployer.*.kubernetes.limits.cpu":"2000m",
"deployer.*.memory":"500Mi",
"deployer.*.kubernetes.limits.memory":"1000Mi",
"app.log.name":"even-log"
}' | jq

$ curl -X POST http://scdf-server.weather.svc.cd1.io.navercorp.com:8088/streams/deployments/rest-test-router-odd-log \
-H "Content-Type: application/json" \
--data '{
"deployer.*.cpu":"100m",
"deployer.*.kubernetes.limits.cpu":"2000m",
"deployer.*.memory":"500Mi",
"deployer.*.kubernetes.limits.memory":"1000Mi",
"app.log.name":"odd-log"
}' | jq
```  

다시 `rest` 이름의 스트림들을 조회하면 아래와 같다.  


```bash

```  

### 실행 중 애플리케이션 확인 (/runtime/apps)
`GET /runtime/apps`
`GET /runtime/apps/{appIp}/instance`

### 스트림 정보 관련 확인 

### 스트림 업데이트 

### 스트림 스케일 조정

### 스트림 배포 취소

### 스트림 삭제



---  
## Reference
[](https://docs.spring.io/spring-cloud-dataflow/docs/2.10.3/reference/htmlsingle/#api-guide)  
