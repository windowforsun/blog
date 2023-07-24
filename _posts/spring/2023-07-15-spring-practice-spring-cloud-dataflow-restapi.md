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
$ curl -X DELETE http://localhost:9393/apps/processor/uppercase-transformer-kafka | jq

.. 조회를 통해 삭제 확인 ..
$ curl -X GET http://localhost:9393/apps\?search\=uppercase\&type\=processor | jq
{
  "_links": {
    "self": {
      "href": "http://localhost:9393/apps?search=uppercase&type=processor&page=0&size=20"
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
$ curl -X POST http://localhost:9393/streams/definitions \
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
      "href": "http://localhost:9393/streams/definitions/rest-time-transform-router"
    }
  }
}

$ curl -X POST http://localhost:9393/streams/definitions \
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
      "href": "http://localhost:9393/streams/definitions/rest-test-router-even-log"
    }
  }
}

$ curl -X POST http://localhost:9393/streams/definitions \
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
      "href": "http://localhost:9393/streams/definitions/rest-test-router-odd-log"
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
$ curl -X GET http://localhost:9393/streams/definitions\?page\=0\&sort\=name,ASC\&search\=rest\&size\=50 | jq

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
            "href": "http://localhost:9393/streams/definitions/rest-test-router-even-log"
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
            "href": "http://localhost:9393/streams/definitions/rest-test-router-odd-log"
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
            "href": "http://localhost:9393/streams/definitions/rest-time-transform-router"
          }
        }
      }
    ]
  },
  "_links": {
    "self": {
      "href": "http://localhost:9393/streams/definitions?search=rest&page=0&size=50&sort=name,asc"
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
$ curl -X POST http://localhost:9393/streams/deployments/rest-time-transform-router \
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

$ curl -X POST http://localhost:9393/streams/deployments/rest-test-router-even-log \
-H "Content-Type: application/json" \
--data '{
"deployer.*.cpu":"100m",
"deployer.*.kubernetes.limits.cpu":"2000m",
"deployer.*.memory":"500Mi",
"deployer.*.kubernetes.limits.memory":"1000Mi",
"app.log.name":"even-log"
}' | jq

$ curl -X POST http://localhost:9393/streams/deployments/rest-test-router-odd-log \
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
$ curl -X GET http://localhost:9393/streams/definitions\?page\=0\&sort\=name,ASC\&search\=rest\&size\=50 | jq
{
  "_embedded": {
    "streamDefinitionResourceList": [
      {
        "name": "rest-test-router-even-log",
        "dslText": ":test-router-even > log --wavefront.application.service=${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown}-${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}} --management.metrics.tags.application.type=${spring.cloud.dataflow.stream.app.type:unknown} --log.name=even-log --management.metrics.tags.stream.name=${spring.cloud.dataflow.stream.name:unknown} --management.metrics.tags.application=${spring.cloud.dataflow.stream.name:unknown}-${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown} --management.metrics.tags.application.guid=${spring.cloud.application.guid:unknown} --management.metrics.tags.application.name=${vcap.application.application_name:${spring.cloud.dataflow.stream.app.label:unknown}} --management.metrics.tags.instance.index=${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}} --wavefront.application.name=${spring.cloud.dataflow.stream.name:unknown}",
        "originalDslText": ":test-router-even > log",
        "status": "deployed",
        "description": "",
        "statusDescription": "The stream has been successfully deployed",
        "_links": {
          "self": {
            "href": "http://localhost:9393/streams/definitions/rest-test-router-even-log"
          }
        }
      },
      {
        "name": "rest-test-router-odd-log",
        "dslText": ":test-router-odd > log --wavefront.application.service=${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown}-${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}} --management.metrics.tags.application.type=${spring.cloud.dataflow.stream.app.type:unknown} --log.name=odd-log --management.metrics.tags.stream.name=${spring.cloud.dataflow.stream.name:unknown} --management.metrics.tags.application=${spring.cloud.dataflow.stream.name:unknown}-${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown} --management.metrics.tags.application.guid=${spring.cloud.application.guid:unknown} --management.metrics.tags.application.name=${vcap.application.application_name:${spring.cloud.dataflow.stream.app.label:unknown}} --management.metrics.tags.instance.index=${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}} --wavefront.application.name=${spring.cloud.dataflow.stream.name:unknown}",
        "originalDslText": ":test-router-odd > log",
        "status": "deployed",
        "description": "",
        "statusDescription": "The stream has been successfully deployed",
        "_links": {
          "self": {
            "href": "http://localhost:9393/streams/definitions/rest-test-router-odd-log"
          }
        }
      },
      {
        "name": "rest-time-transform-router",
        "dslText": "time --management.metrics.tags.application.type=${spring.cloud.dataflow.stream.app.type:unknown} --management.metrics.tags.stream.name=${spring.cloud.dataflow.stream.name:unknown} --management.metrics.tags.application=${spring.cloud.dataflow.stream.name:unknown}-${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown} --time.date-format='yyyy-MM-dd HH:mm:ss' --management.metrics.tags.instance.index=${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}} --wavefront.application.service=${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown}-${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}} --spring.integration.poller.fixed-rate=1000 --management.metrics.tags.application.guid=${spring.cloud.application.guid:unknown} --management.metrics.tags.application.name=${vcap.application.application_name:${spring.cloud.dataflow.stream.app.label:unknown}} --wavefront.application.name=${spring.cloud.dataflow.stream.name:unknown} | transform --management.metrics.tags.application.type=${spring.cloud.dataflow.stream.app.type:unknown} --management.metrics.tags.stream.name=${spring.cloud.dataflow.stream.name:unknown} --management.metrics.tags.application=${spring.cloud.dataflow.stream.name:unknown}-${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown} --management.metrics.tags.instance.index=${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}} --wavefront.application.service=${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown}-${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}} --spel.function.expression='payload.substring(payload.length() - 1)' --management.metrics.tags.application.guid=${spring.cloud.application.guid:unknown} --management.metrics.tags.application.name=${vcap.application.application_name:${spring.cloud.dataflow.stream.app.label:unknown}} --wavefront.application.name=${spring.cloud.dataflow.stream.name:unknown} | router --wavefront.application.service=${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown}-${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}} --management.metrics.tags.application.type=${spring.cloud.dataflow.stream.app.type:unknown} --management.metrics.tags.stream.name=${spring.cloud.dataflow.stream.name:unknown} --management.metrics.tags.application=${spring.cloud.dataflow.stream.name:unknown}-${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown} --router.expression='new Integer(payload) % 2 == 0 ? test-router-even : test-router-odd' --management.metrics.tags.application.guid=${spring.cloud.application.guid:unknown} --management.metrics.tags.application.name=${vcap.application.application_name:${spring.cloud.dataflow.stream.app.label:unknown}} --management.metrics.tags.instance.index=${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}} --wavefront.application.name=${spring.cloud.dataflow.stream.name:unknown}",
        "originalDslText": "time | transform | router",
        "status": "deployed",
        "description": "",
        "statusDescription": "The stream has been successfully deployed",
        "_links": {
          "self": {
            "href": "http://localhost:9393/streams/definitions/rest-time-transform-router"
          }
        }
      }
    ]
  },
  "_links": {
    "self": {
      "href": "http://localhost:9393/streams/definitions?search=rest&page=0&size=50&sort=name,asc"
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

상태값을 의미하는 `status` 필드가 `deployed` 로 변경됐고, 
배포때 사용한 프로퍼티 값이 `dslText` 에 설정돼 있는 것을 확인 할 수 있다.  

### 실행 중 애플리케이션 확인 (/runtime/apps)
`GET /runtime/apps` 요청은 현재 `SCDF` 를 통해 실행 중인 전체 애플리케이션을 조회할 수 있다.  

```bash
$ curl -X GET http://localhost:9393/runtime/apps | jq
{
  "_embedded": {
    "appStatusResourceList": [
      {
        "deploymentId": "rest-test-router-even-log-log-v1",
        "state": "deployed",
        "instances": {
          "_embedded": {
            "appInstanceStatusResourceList": [
              {
                "instanceId": "rest-test-router-even-log-log-v1-699696646c-psxd2",
                "state": "deployed",
                "attributes": {
                  "container.restartCount": "0",
                  "guid": "3bb16063-b09d-4eeb-8414-caf499af6252",
                  "host.ip": "10.172.113.207",
                  "phase": "Running",
                  "pod.ip": "10.174.131.110",
                  "pod.name": "rest-test-router-even-log-log-v1-699696646c-psxd2",
                  "pod.startTime": "2023-07-16T11:06:48Z",
                  "service.name": "rest-test-router-even-log-log",
                  "skipper.application.name": "log",
                  "skipper.release.name": "rest-test-router-even-log",
                  "skipper.release.version": "1",
                  "spring.app.id": "rest-test-router-even-log-log-v1",
                  "spring.deployment.id": "rest-test-router-even-log-log-v1"
                },
                "guid": "3bb16063-b09d-4eeb-8414-caf499af6252",
                "_links": {
                  "self": {
                    "href": "http://localhost:9393/runtime/apps/rest-test-router-even-log-log-v1/instances/rest-test-router-even-log-log-v1-699696646c-psxd2"
                  }
                }
              }
            ]
          }
        },
        "name": "log",
        "_links": {
          "self": {
            "href": "http://localhost:9393/runtime/apps/rest-test-router-even-log-log-v1"
          }
        }
      },
      {
        "deploymentId": "rest-test-router-odd-log-log-v1",
        "state": "deployed",
        "instances": {
          "_embedded": {
            "appInstanceStatusResourceList": [
              {
                "instanceId": "rest-test-router-odd-log-log-v1-85b675f4dd-tcpsg",
                "state": "deployed",
                "attributes": {
                  "container.restartCount": "0",
                  "guid": "9e5b4336-5116-43c5-a6b4-9288fd115c77",
                  "host.ip": "10.172.113.207",
                  "phase": "Running",
                  "pod.ip": "10.174.131.122",
                  "pod.name": "rest-test-router-odd-log-log-v1-85b675f4dd-tcpsg",
                  "pod.startTime": "2023-07-16T11:06:53Z",
                  "service.name": "rest-test-router-odd-log-log",
                  "skipper.application.name": "log",
                  "skipper.release.name": "rest-test-router-odd-log",
                  "skipper.release.version": "1",
                  "spring.app.id": "rest-test-router-odd-log-log-v1",
                  "spring.deployment.id": "rest-test-router-odd-log-log-v1"
                },
                "guid": "9e5b4336-5116-43c5-a6b4-9288fd115c77",
                "_links": {
                  "self": {
                    "href": "http://localhost:9393/runtime/apps/rest-test-router-odd-log-log-v1/instances/rest-test-router-odd-log-log-v1-85b675f4dd-tcpsg"
                  }
                }
              }
            ]
          }
        },
        "name": "log",
        "_links": {
          "self": {
            "href": "http://localhost:9393/runtime/apps/rest-test-router-odd-log-log-v1"
          }
        }
      },
      {
        "deploymentId": "rest-time-transform-router-router-v2",
        "state": "deployed",
        "instances": {
          "_embedded": {
            "appInstanceStatusResourceList": [
              {
                "instanceId": "rest-time-transform-router-router-v2-6c897bdf49-7nplc",
                "state": "deployed",
                "attributes": {
                  "container.restartCount": "0",
                  "guid": "f9129c64-1639-4c56-96e1-a0edaf86ba82",
                  "host.ip": "10.174.53.153",
                  "phase": "Running",
                  "pod.ip": "10.175.40.81",
                  "pod.name": "rest-time-transform-router-router-v2-6c897bdf49-7nplc",
                  "pod.startTime": "2023-07-16T11:06:33Z",
                  "service.name": "rest-time-transform-router-router",
                  "skipper.application.name": "router",
                  "skipper.release.name": "rest-time-transform-router",
                  "skipper.release.version": "2",
                  "spring.app.id": "rest-time-transform-router-router-v2",
                  "spring.deployment.id": "rest-time-transform-router-router-v2"
                },
                "guid": "f9129c64-1639-4c56-96e1-a0edaf86ba82",
                "_links": {
                  "self": {
                    "href": "http://localhost:9393/runtime/apps/rest-time-transform-router-router-v2/instances/rest-time-transform-router-router-v2-6c897bdf49-7nplc"
                  }
                }
              }
            ]
          }
        },
        "name": "router",
        "_links": {
          "self": {
            "href": "http://localhost:9393/runtime/apps/rest-time-transform-router-router-v2"
          }
        }
      },
      {
        "deploymentId": "rest-time-transform-router-transform-v2",
        "state": "deployed",
        "instances": {
          "_embedded": {
            "appInstanceStatusResourceList": [
              {
                "instanceId": "rest-time-transform-router-transform-v2-bb755c845-4rz6s",
                "state": "deployed",
                "attributes": {
                  "container.restartCount": "0",
                  "guid": "75764bbd-fd76-43a1-9145-f02c6bb51a6d",
                  "host.ip": "10.113.134.10",
                  "phase": "Running",
                  "pod.ip": "10.175.6.151",
                  "pod.name": "rest-time-transform-router-transform-v2-bb755c845-4rz6s",
                  "pod.startTime": "2023-07-16T11:06:33Z",
                  "service.name": "rest-time-transform-router-transform",
                  "skipper.application.name": "transform",
                  "skipper.release.name": "rest-time-transform-router",
                  "skipper.release.version": "2",
                  "spring.app.id": "rest-time-transform-router-transform-v2",
                  "spring.deployment.id": "rest-time-transform-router-transform-v2"
                },
                "guid": "75764bbd-fd76-43a1-9145-f02c6bb51a6d",
                "_links": {
                  "self": {
                    "href": "http://localhost:9393/runtime/apps/rest-time-transform-router-transform-v2/instances/rest-time-transform-router-transform-v2-bb755c845-4rz6s"
                  }
                }
              }
            ]
          }
        },
        "name": "transform",
        "_links": {
          "self": {
            "href": "http://localhost:9393/runtime/apps/rest-time-transform-router-transform-v2"
          }
        }
      },
      {
        "deploymentId": "rest-time-transform-router-time-v2",
        "state": "deployed",
        "instances": {
          "_embedded": {
            "appInstanceStatusResourceList": [
              {
                "instanceId": "rest-time-transform-router-time-v2-667d54d7cd-tbwc6",
                "state": "deployed",
                "attributes": {
                  "container.restartCount": "0",
                  "guid": "a950d44c-2a8d-49ff-9edd-422682e9a531",
                  "host.ip": "10.172.113.206",
                  "phase": "Running",
                  "pod.ip": "10.174.130.193",
                  "pod.name": "rest-time-transform-router-time-v2-667d54d7cd-tbwc6",
                  "pod.startTime": "2023-07-16T11:06:33Z",
                  "service.name": "rest-time-transform-router-time",
                  "skipper.application.name": "time",
                  "skipper.release.name": "rest-time-transform-router",
                  "skipper.release.version": "2",
                  "spring.app.id": "rest-time-transform-router-time-v2",
                  "spring.deployment.id": "rest-time-transform-router-time-v2"
                },
                "guid": "a950d44c-2a8d-49ff-9edd-422682e9a531",
                "_links": {
                  "self": {
                    "href": "http://localhost:9393/runtime/apps/rest-time-transform-router-time-v2/instances/rest-time-transform-router-time-v2-667d54d7cd-tbwc6"
                  }
                }
              }
            ]
          }
        },
        "name": "time",
        "_links": {
          "self": {
            "href": "http://localhost:9393/runtime/apps/rest-time-transform-router-time-v2"
          }
        }
      },
    ]
  },
  "_links": {
    "self": {
      "href": "http://localhost:9393/runtime/apps?page=0&size=20"
    }
  },
  "page": {
    "size": 20,
    "totalElements": 5,
    "totalPages": 1,
    "number": 0
  }
}
```  

`rest-time-transform-router`, `rest-test-router-even`, `rest-test-router-odd` 스트림에서 
실행 중인 5개 애플리케이션이 조회되는 것을 확인 할 수 있다.  

`GET /runtime/apps/{appIp}/instance` 요청은 실행 중인 전체 애플리케이션 중 
특정 애플리케이션에 대한 인스턴스 정보를 조회할 수 있다. 
테스트를 위해 시간값을 1초마다 생성하는 `rest-time-transform-router` 스트림 중 `time` 애플리케이션을 조회 하면 아래와 같다.  

```bash
$ curl -X GET http://localhost:9393/runtime/apps/rest-time-transform-router-time-v2/instances | jq 
{
  "_embedded": {
    "appInstanceStatusResourceList": [
      {
        "instanceId": "rest-time-transform-router-time-v2-667d54d7cd-tbwc6",
        "state": "deployed",
        "attributes": {
          "container.restartCount": "0",
          "guid": "a950d44c-2a8d-49ff-9edd-422682e9a531",
          "host.ip": "10.172.113.206",
          "phase": "Running",
          "pod.ip": "10.174.130.193",
          "pod.name": "rest-time-transform-router-time-v2-667d54d7cd-tbwc6",
          "pod.startTime": "2023-07-16T11:06:33Z",
          "service.name": "rest-time-transform-router-time",
          "skipper.application.name": "time",
          "skipper.release.name": "rest-time-transform-router",
          "skipper.release.version": "2",
          "spring.app.id": "rest-time-transform-router-time-v2",
          "spring.deployment.id": "rest-time-transform-router-time-v2"
        },
        "guid": "a950d44c-2a8d-49ff-9edd-422682e9a531",
        "_links": {
          "self": {
            "href": "http://localhost:9393/runtime/apps/rest-time-transform-router-time-v2/instances/rest-time-transform-router-time-v2-667d54d7cd-tbwc6"
          }
        }
      }
    ]
  },
  "_links": {
    "self": {
      "href": "http://localhost:9393/runtime/apps/rest-time-transform-router-time-v2/instances?page=0&size=20"
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

### 스트림 정보 관련 확인 (/streams/definitions/{name})
`GET /streams/definitions/{name}` 요청은 특정 스트림의 상세 정보를 조회 할 수 있다.  

```bash
$ curl -X GET http://localhost:9393/streams/definitions/rest-time-transform-router | jq
{
  "name": "rest-time-transform-router",
  "dslText": "time --management.metrics.tags.application.type=${spring.cloud.dataflow.stream.app.type:unknown} --management.metrics.tags.stream.name=${spring.cloud.dataflow.stream.name:unknown} --management.metrics.tags.application=${spring.cloud.dataflow.stream.name:unknown}-${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown} --time.date-format='yyyy-MM-dd HH:mm:ss' --management.metrics.tags.instance.index=${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}} --wavefront.application.service=${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown}-${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}} --spring.integration.poller.fixed-rate=1000 --management.metrics.tags.application.guid=${spring.cloud.application.guid:unknown} --management.metrics.tags.application.name=${vcap.application.application_name:${spring.cloud.dataflow.stream.app.label:unknown}} --wavefront.application.name=${spring.cloud.dataflow.stream.name:unknown} | transform --management.metrics.tags.application.type=${spring.cloud.dataflow.stream.app.type:unknown} --management.metrics.tags.stream.name=${spring.cloud.dataflow.stream.name:unknown} --management.metrics.tags.application=${spring.cloud.dataflow.stream.name:unknown}-${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown} --management.metrics.tags.instance.index=${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}} --wavefront.application.service=${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown}-${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}} --spel.function.expression='payload.substring(payload.length() - 1)' --management.metrics.tags.application.guid=${spring.cloud.application.guid:unknown} --management.metrics.tags.application.name=${vcap.application.application_name:${spring.cloud.dataflow.stream.app.label:unknown}} --wavefront.application.name=${spring.cloud.dataflow.stream.name:unknown} | router --wavefront.application.service=${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown}-${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}} --management.metrics.tags.application.type=${spring.cloud.dataflow.stream.app.type:unknown} --management.metrics.tags.stream.name=${spring.cloud.dataflow.stream.name:unknown} --management.metrics.tags.application=${spring.cloud.dataflow.stream.name:unknown}-${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown} --router.expression='new Integer(payload) % 2 == 0 ? test-router-even : test-router-odd' --management.metrics.tags.application.guid=${spring.cloud.application.guid:unknown} --management.metrics.tags.application.name=${vcap.application.application_name:${spring.cloud.dataflow.stream.app.label:unknown}} --management.metrics.tags.instance.index=${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}} --wavefront.application.name=${spring.cloud.dataflow.stream.name:unknown}",
  "originalDslText": "time | transform | router",
  "status": "deployed",
  "description": "",
  "statusDescription": "The stream has been successfully deployed",
  "_links": {
    "self": {
      "href": "http://localhost:9393/streams/definitions/rest-time-transform-router"
    }
  }
}
```  

### 스트림 업데이트 (/streams/deployments/update/{name})
`POST /streams/deployments/update/{name}` 요청과 함께 `Request Body` 에 
업데이트가 필요한 프로퍼티를 함께 사용하면 기존 실행 중인 애플리케이션을 업데이트 할 수 있다.  

아래는 `rest-time-transform-router` 스트림의 `time` 애플리케이션의 주기를 `1초 -> 0.1초`로 변경하고, 
출력 포맷은 `yyyy-MM-dd HH:mm:ss -> yyyyMMddHHmmssSSS` 로 변경한 예시이다.  

```bash
$ curl -X POST http://localhost:9393/streams/deployments/update/rest-time-transform-router \
-H "Content-Type: application/json" \
--data '{
"releaseName" : "rest-time-transform-router",
    "packageIdentifier" : {
          "repositoryName" : "test",
          "packageName" : "rest-time-transform-router",
          "packageVersion" : "1.0.0"
      },
    "updateProperties" : {
        "app.time.spring.integration.poller.fixed-rate":100,
        "app.time.date-format":"yyyyMMddHHmmssSSS"
      }
    }
}'
```  

업데이트 확인을 위해 `rest-time-transform-router` 스트림의 상세 정보를 조회하면 아래와 같이 
업데이트를 요청한 값과 동일하게 반영된 것을 확인 할 수 있다.  

```bash
$ curl -X GET http://localhost:9393/streams/definitions/rest-time-transform-router | jq
{
  "name": "rest-time-transform-router",
  "dslText": "time --time.date-format=yyyyMMddHHmmssSSS --spring.integration.poller.fixed-rate=100 --management.metrics.tags.application.guid=${spring.cloud.application.guid:unknown} --management.metrics.tags.application.name=${vcap.application.application_name:${spring.cloud.dataflow.stream.app.label:unknown}} --wavefront.application.name=${spring.cloud.dataflow.stream.name:unknown} | transform --management.metrics.tags.application.type=${spring.cloud.dataflow.stream.app.type:unknown} --management.metrics.tags.stream.name=${spring.cloud.dataflow.stream.name:unknown} --management.metrics.tags.application=${spring.cloud.dataflow.stream.name:unknown}-${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown} --management.metrics.tags.instance.index=${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}} --wavefront.application.service=${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown}-${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}} --spel.function.expression='payload.substring(payload.length() - 1)' --management.metrics.tags.application.guid=${spring.cloud.application.guid:unknown} --management.metrics.tags.application.name=${vcap.application.application_name:${spring.cloud.dataflow.stream.app.label:unknown}} --wavefront.application.name=${spring.cloud.dataflow.stream.name:unknown} | router --wavefront.application.service=${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown}-${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}} --management.metrics.tags.application.type=${spring.cloud.dataflow.stream.app.type:unknown} --management.metrics.tags.stream.name=${spring.cloud.dataflow.stream.name:unknown} --management.metrics.tags.application=${spring.cloud.dataflow.stream.name:unknown}-${spring.cloud.dataflow.stream.app.label:unknown}-${spring.cloud.dataflow.stream.app.type:unknown} --router.expression='new Integer(payload) % 2 == 0 ? test-router-even : test-router-odd' --management.metrics.tags.application.guid=${spring.cloud.application.guid:unknown} --management.metrics.tags.application.name=${vcap.application.application_name:${spring.cloud.dataflow.stream.app.label:unknown}} --management.metrics.tags.instance.index=${vcap.application.instance_index:${spring.cloud.stream.instanceIndex:0}} --wavefront.application.name=${spring.cloud.dataflow.stream.name:unknown}",
  "originalDslText": "time | transform | router",
  "status": "deployed",
  "description": "",
  "statusDescription": "The stream has been successfully deployed",
  "_links": {
    "self": {
      "href": "http://localhost:9393/streams/definitions/rest-time-transform-router"
    }
  }
}
```  


### 스트림 스케일 조정 (/streams/deployments/scale)
`POST /streams/deployments/scale/{streamName}/{appName}/instance/{count}` 
요청을 통해서는 스트림을 구성하는 특정 애플리케이션의 스케일을 조정할 수 있다.  

아래는 `rest-time-transform-router` 스트림의 `time` 애플리케이션의 스케일을 `2` 조정하는 예시이다.  

```bash
$ curl -X POST http://localhost:9393/streams/deployments/scale/rest-time-transform-router/time/instances/2 | jq

```  

위 요청을 수행한 뒤 `rest-time-transform-router` 스트림의 `time` 애플리케이션을 조회하면 아래와 같이 2개의 인스턴스가 조회되는 것을 확인 할 수 있다.  

```bash
$ curl -X GET http://localhost:9393/runtime/apps/rest-time-transform-router-time-v3/instances | jq
{
  "_embedded": {
    "appInstanceStatusResourceList": [
      {
        "instanceId": "rest-time-transform-router-time-v3-7d85cc49b8-8gg6h",
        "state": "deployed",
        "attributes": {
          "container.restartCount": "0",
          "guid": "7ea6b44d-4a75-42ab-96e6-48b1ce5e3b25",
          "host.ip": "10.174.53.157",
          "phase": "Running",
          "pod.ip": "10.175.42.65",
          "pod.name": "rest-time-transform-router-time-v3-7d85cc49b8-8gg6h",
          "pod.startTime": "2023-07-23T09:21:12Z",
          "service.name": "rest-time-transform-router-time",
          "skipper.application.name": "time",
          "skipper.release.name": "rest-time-transform-router",
          "skipper.release.version": "3",
          "spring.app.id": "rest-time-transform-router-time-v3",
          "spring.deployment.id": "rest-time-transform-router-time-v3"
        },
        "guid": "7ea6b44d-4a75-42ab-96e6-48b1ce5e3b25",
        "_links": {
          "self": {
            "href": "http://localhost:9393/runtime/apps/rest-time-transform-router-time-v3/instances/rest-time-transform-router-time-v3-7d85cc49b8-8gg6h"
          }
        }
      },
      {
        "instanceId": "rest-time-transform-router-time-v3-7d85cc49b8-wwpxf",
        "state": "deployed",
        "attributes": {
          "container.restartCount": "0",
          "guid": "ceb87165-907b-4f6e-822f-6fb62de5e4f3",
          "host.ip": "10.174.53.138",
          "phase": "Running",
          "pod.ip": "10.175.32.250",
          "pod.name": "rest-time-transform-router-time-v3-7d85cc49b8-wwpxf",
          "pod.startTime": "2023-07-23T09:34:53Z",
          "service.name": "rest-time-transform-router-time",
          "skipper.application.name": "time",
          "skipper.release.name": "rest-time-transform-router",
          "skipper.release.version": "3",
          "spring.app.id": "rest-time-transform-router-time-v3",
          "spring.deployment.id": "rest-time-transform-router-time-v3"
        },
        "guid": "ceb87165-907b-4f6e-822f-6fb62de5e4f3",
        "_links": {
          "self": {
            "href": "http://localhost:9393/runtime/apps/rest-time-transform-router-time-v3/instances/rest-time-transform-router-time-v3-7d85cc49b8-wwpxf"
          }
        }
      }
    ]
  },
  "_links": {
    "self": {
      "href": "http://localhost:9393/runtime/apps/rest-time-transform-router-time-v3/instances?page=0&size=20"
    }
  },
  "page": {
    "size": 20,
    "totalElements": 2,
    "totalPages": 1,
    "number": 0
  }
}
```  

### 스트림 로그 (/streams/log)
`GET /streams/logs/{streamName}` 요청은 스트림을 구성하는 전체 애플리케이션들의 로그를 한번에 조회 할 수 있다.  

`rest-time-transform-router` 스트림의 로그를 조회하면 아래와 같다.

```bash
$ curl -X GET http://localhost:9393/streams/logs/rest-time-transform-router | jq
{
  "logs": {
    "rest-time-transform-router-transform-v2": "... log output .....",
    "rest-time-transform-router-router-v2": "... log output .....",
    "rest-time-transform-router-time-v3": "... log output ....."
  }
}
```  

`GET /streams/logs/{streamName}/{appName}` 요청은 스트림을 구성하는 특정 애플리케이션의 로그를 확인 할 수 있다.  

`rest-test-router-even-log` 스트림의 `log` 애플리케이션의 로그를 조회하면 아래와 같다.  

```bash
$ curl -X GET http://localhost:9393/streams/logs/rest-test-router-even-log/rest-test-router-even-log-log-v1 | jq
{
  "logs": {
    "rest-test-router-even-log-log-v1": "2023-07-16 20:07:20.257  INFO [log-sink,f7364e11d6268eb6,9ca258be2f55d15c] 6 --- [container-0-C-1] even-log                                 : 6\n2023-07-16 20:07:20.257  INFO [log-sink,0d0b379b11315fee,80dbb90fe7dffecf] 6 --- [container-0-C-1] even-log                                 : 6\n"
  }
}

```  

### 스트림 배포 취소 (/streams/deployments/{streamName})
`DELETE /streams/deploymenets/{streamName}` 요청은 배포한 스트림 구성을 다시 취소할 수 있다.  

```bash
$ curl -X DELETE http://localhost:9393/streams/deployments/rest-time-transform-router | jq 

$ curl -X DELETE http://localhost:9393/streams/deployments/rest-test-router-even-log | jq 

$ curl -X DELETE http://localhost:9393/streams/deployments/rest-test-router-odd-log | jq
```  

다시 `rest` 로 시작하는 스트림을 조회하면 아래와 같이 상태가 `undeployed`로 변경된 것을 확인 할 수 있다.  

```bash
$ curl -X GET http://localhost:9393/streams/definitions\?page\=0\&sort\=name,ASC\&search\=rest\&size\=50 | jq
{
  "_embedded": {
    "streamDefinitionResourceList": [
      {
        "name": "rest-test-router-even-log",
        "dslText": ".....",
        "originalDslText": ":test-router-even > log",
        "status": "undeployed",
        "description": "",
        "statusDescription": "The app or group is known to the system, but is not currently deployed",
        "_links": {
          "self": {
            "href": "http://localhost:9393/streams/definitions/rest-test-router-even-log"
          }
        }
      },
      {
        "name": "rest-test-router-odd-log",
        "dslText": ".....",
        "originalDslText": ":test-router-odd > log",
        "status": "undeployed",
        "description": "",
        "statusDescription": "The app or group is known to the system, but is not currently deployed",
        "_links": {
          "self": {
            "href": "http://localhost:9393/streams/definitions/rest-test-router-odd-log"
          }
        }
      },
      {
        "name": "rest-time-transform-router",
        "dslText": ".....",
        "originalDslText": "time | transform | router",
        "status": "undeployed",
        "description": "",
        "statusDescription": "The app or group is known to the system, but is not currently deployed",
        "_links": {
          "self": {
            "href": "http://localhost:9393/streams/definitions/rest-time-transform-router"
          }
        }
      }
    ]
  },
  "_links": {
    "self": {
      "href": "http://localhost:9393/streams/definitions?search=rest&page=0&size=50&sort=name,asc"
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

### 스트림 삭제 (/streams/definitions/{streamName)
`DELETE /streams/definitions/{streamName}` 요청은 생성한 스트림을 삭제할 수 있다.  

```bash
$ curl -X DELETE http://localhost:9393/streams/definitions/rest-test-router-odd-log | jq

$ curl -X DELETE http://localhost:9393/streams/definitions/rest-test-router-even-log | jq

$ curl -X DELETE http://localhost:9393/streams/definitions/rest-time-transform-router | jq
```  

다시 `rest` 로 시작하는 스트림을 조회하면 아래와 같이 매칭되는 애플리케이션이 조회되지 않는 것을 확인 할 수 있다.  

```bash
$ curl -X GET http://localhost:9393/streams/definitions\?page\=0\&sort\=name,ASC\&search\=rest\&size\=50 | jq
{
  "_links": {
    "self": {
      "href": "http://localhost:9393/streams/definitions?search=rest&page=0&size=50&sort=name,asc"
    }
  },
  "page": {
    "size": 50,
    "totalElements": 0,
    "totalPages": 0,
    "number": 0
  }
}
```  


---  
## Reference
[](https://docs.spring.io/spring-cloud-dataflow/docs/2.10.3/reference/htmlsingle/#api-guide)  
