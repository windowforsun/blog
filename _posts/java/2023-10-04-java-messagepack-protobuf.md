--- 
layout: single
classes: wide
title: "[Java 실습] "
header:
  overlay_image: /img/java-bg.jpg 
excerpt: ''
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
  - Concept
  - Java
  - MessagePack
  - Protobuf
  - Serialize
  - Deserialize
toc: true 
use_math: true
---  

## Serialize
`Serialize` 는 프로그램이 사용하는 객체를 `Data stream` 으로 만드는 것을 의미한다. 
반대로 `Data stream` 을 다시 프로그램이 사용할 수 있는 객체로 변환하는 것을 `Deserialize` 라고 한다.  

이러한 직렬화와 역직렬화가 필요한 이유는 객체 자체의 영속적으로 보관할 때 파일형대태로 만들어 저장소에 저장하거나, 
네트워크를 통해 다른 엔드포인트로 전달이 가능하다.  

직렬화에는 많은 방식들이 있지만, 서버-클라이언트 사이의 데이터 송수신에는 주로 `Json` 이 사용된다. 
`Json` 도 `HTML`, `XML` 등을 거치면서 단점을 보완한 직렬화 포멧이지만, 
이 또한 데이터 크기나 직렬화 퍼포먼스등의 단점이 존재한다.  

그래서 이번 포스트에서는 `Java` 언에서 `Json` 을 대신해서 사용할 수 있는 직렬화 포멧인 `MessagePack` 과 `Protobuf` 에 대해 알아본다.  

### MessagePack
`MessagePack` 은 `C#` 용 직렬화 포멧으로 다양한 언어에서도 사용할 수 있도록 라이브러리를 제공한다. 
`Json` 은 직렬화 결과가 문자열 스트림인 것에 비해 `MessagePack` 은 `byte` 형태이면서 별도의 `metadata` 가 
붙지 않아 `Json` 에 비해 용량 절검에 이득이 있다. 
또한 직렬화/역직렬화 생능 또한 `Json` 과 비교 했을 때 비교적 이점이 있다고 한다. 
자세한 내용은 [MessagePack](https://msgpack.org/)
에서 확인 가능하다.  

`MessagePack` 을 사용하는 방법은 `Jackson json` 라이브러리가 포함된 상태에서 
아래 의존성만 추가해주면 된다.  

```groovy
dependencies {
    // json
    implementation group: 'com.fasterxml.jackson.core', name: 'jackson-databind', version: '2.15.2'

    // message pack
    implementation group: 'org.msgpack', name: 'jackson-dataformat-msgpack', version: '0.9.5'
}
```  

사용법은 기존 `Jackson` 라이브러리의 인터페이스와 동일하다.  

```java
// 저장할 객체 생성
ExamModel examModel = ExamModel.builder()
        .innerModelList(list)
        .innerModelMap(map)
        .build();

// 직렬화 라이브러리 객체 초기화
ObjectMapper objectMapper = new MessagePackMapper();

// 직렬화
byte[] msgPackBytes = objectMapper.writeValueAsBytes(examModel);

// 역직렬화
ExamModel decode = objectMapper.readValue(msgPackBytes, ExamModel.class);
```  

### Protobuf
`Protobuf` 는 `Protocol Buffers` 의 약자로 `Google` 에서 개발한 데이터 직렬화 포멧이다. 
구조화된 데이터를 `byte` 형태로 효율적이고 컴팩트하게 저장해서 네트워크를 통해 더 빠르게 전송이 가능하다. 
`Protobuf` 또한 다양한 언어를 지원하는 라이브러리가 있어 용이하게 사용 할 수 있다. 
`Json` 과 비교해서 구조화된 `byte` 형식으로 더 적은 용량으로 결과를 만들어 낼 수 있고, 
직렬화/역직렬화도 더 빠른 성능을 보여준다.  

하지만 구조화된 `byte` 형태를 만들기 위해서, `Protobuf` 을 위한 별도의 스키마 파일을 구성해야 한다는 
불편한 점이 있다. 
자세한 내용은 [Protobuf](https://protobuf.dev/)
에서 확인 가능하다.  

`Java` 에서 `Protobuf` 를 사용하기 위해서는 아래와 같은 과정을 거쳐야 한다.  

1. 빌드 도구(`maven`, `gradle`) 에 의존성과 스키마 빌드를 위한 플러그인 추가 및 설정
2. `Protobuf` 스키마 정의
3. 스키마를 `Java Class` 로 빌드

[의존성 가이드](https://github.com/protocolbuffers/protobuf/tree/main/java),
[플러그인 가이드](https://github.com/google/protobuf-gradle-plugin)
를 보고 아래와 같이 `build.gradle` 에 설정을 추가한다.  

```groovy
plugins {
    id 'java'
    id 'com.google.protobuf' version '0.9.4'
}


apply plugin: 'com.google.protobuf'

repositories {
  mavenCentral()
}

dependencies {
  // protobuf
  implementation 'com.google.protobuf:protobuf-java:3.22.3'
  implementation 'com.google.protobuf:protobuf-java-util:3.22.2'
}

// 스키마에 해당하는 Java Class 가 생성될 위치
sourceSets {
  main {
    java {
      srcDirs 'build/generated/source/proto/main/java'
    }
  }
}

protobuf {
  protoc {
    if (osdetector.os == 'osx') {
      artifact = 'com.google.protobuf:protoc:3.22.2:osx-x86_64'
    } else {
      artifact = 'com.google.protobuf:protoc:3.22.2'
    }
  }

  generateProtoTasks {
    all()*.plugins {
      grpc {}
    }
  }
}

test {
  useJUnitPlatform()
}
```  

그리고 `scr/main/proto` 디렉토리를 생성하고, 
그 하위에 `.proto` 확장자로 `Protobuf` 의 스키마를 아래와 같이 정의한다.  

```protobuf
// schema.proto

syntax = "proto3";
package com.windowforsun.serialize.proto;

option java_package = "com.windowforsun.serialize.proto";
option java_outer_classname = "Proto";

message ExamModel {
  repeated ExamInnerModel inner_model_list = 1;
  map<string, ExamInnerModel> inner_model_map = 2;
}

message ExamInnerModel {
  string str = 1;
  int32 intValue = 2;
  double doubleValue = 3;

}
```  

`Protobuf` 스키마 작성을 위한 가이드는 [여기](https://protobuf.dev/programming-guides/proto3/)
를 참고한다.  

이제 빌드 도구를 사용해서 스키마를 `Java Class` 로 빌드한다.  

```bash
$ ./gradlew generateProto

BUILD SUCCESSFUL in 1s
6 actionable tasks: 6 executed
```  

빌드에 성공하면 `build.gradle` 에 설정한 경로에 스키마에 해당하는 `Java Class` 가 만들어지는데, 
해당 클래스를 사용해서 직렬화/역질렬화가 가능하다.  

```java
// 일반 도메인 객체를 Protobuf 객체로 변환하도록 구현한 메소드

public Proto.ExamModel toProto() {
    return Proto.ExamModel.newBuilder()
            .addAllInnerModelList(Objects.requireNonNullElse(this.innerModelList, Collections.<ExamInnerModel>emptyList())
                    .stream()
                    .map(ExamInnerModel::toProto)
                    .collect(Collectors.toList()))
            .putAllInnerModelMap(Objects.requireNonNullElse(this.innerModelMap, Collections.<String, ExamInnerModel>emptyMap())
                    .entrySet()
                    .stream()
                    .collect(Collectors.<Map.Entry<String, ExamInnerModel>, String, Proto.ExamInnerModel>toMap(
                            Map.Entry::getKey,
                            entry -> entry.getValue().toProto()
                    )))
            .build();
}

// 도메인 객체 생성
ExamModel examModel = ExamModel.builder()
        .innerModelList(list)
        .innerModelMap(map)
        .build();

// 도메인 객체를 Protobuf 객체로 변환
Proto.ExamModel examModelProto = examModel.toProto();

// 직렬화
byte[] protoBytes = examModelProto.toByteArray();
        
// 역직렬화
Proto.ExamModel decode = Proto.ExamModel.parseFrom(protoBytes);
```  


### Comparison of Json, MessagePack, Protobuf
테스트로 사용할 도메인 객체의 클래스는 아래와 같다.  


```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ExamModel {
    private List<ExamInnerModel> innerModelList;
    private Map<String, ExamInnerModel> innerModelMap;

    public Proto.ExamModel toProto() {
        return Proto.ExamModel.newBuilder()
                .addAllInnerModelList(Objects.requireNonNullElse(this.innerModelList, Collections.<ExamInnerModel>emptyList())
                        .stream()
                        .map(ExamInnerModel::toProto)
                        .collect(Collectors.toList()))
                .putAllInnerModelMap(Objects.requireNonNullElse(this.innerModelMap, Collections.<String, ExamInnerModel>emptyMap())
                        .entrySet()
                        .stream()
                        .collect(Collectors.<Map.Entry<String, ExamInnerModel>, String, Proto.ExamInnerModel>toMap(
                                Map.Entry::getKey,
                                entry -> entry.getValue().toProto()
                        )))
                .build();
    }
}

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ExamInnerModel {
  private String str;
  private int intValue;
  private double doubleValue;

  public Proto.ExamInnerModel toProto() {
    return Proto.ExamInnerModel.newBuilder()
            .setStr(this.str)
            .setIntValue(this.intValue)
            .setDoubleValue(this.doubleValue)
            .build();
  }
}
```  

앞서 설명이 됐지만, 위 클래스를 `Protobuf` 스키마로 작성하면 아래와 같다.  

```protobuf
// schema.proto

syntax = "proto3";
package com.windowforsun.serialize.proto;

option java_package = "com.windowforsun.serialize.proto";
option java_outer_classname = "Proto";

message ExamModel {
  repeated ExamInnerModel inner_model_list = 1;
  map<string, ExamInnerModel> inner_model_map = 2;
}

message ExamInnerModel {
  string str = 1;
  int32 intValue = 2;
  double doubleValue = 3;

}
```

#### Data size Comparison
먼저 3개의 직렬화 방식에 대한 크기 비교먼저 진행한다. 
테스트로 사용할 데이터를 생성하는 코드는 아래와 같다.  

```java
public static ExamModel createExamModel() {
    List<ExamInnerModel> list = new ArrayList<>();
    Map<String, ExamInnerModel> map = new HashMap<>();
    Random random = new Random();

    for (int i = 0; i < 10000; i++) {
        ExamInnerModel examInnerModel = ExamInnerModel.builder()
                .str(UUID.randomUUID().toString())
                .intValue(random.nextInt())
                .doubleValue(random.nextDouble())
                .build();
        list.add(examInnerModel);
        map.put(String.valueOf(i), examInnerModel);
    }

    ExamModel examModel = ExamModel.builder()
            .innerModelList(list)
            .innerModelMap(map)
            .build();

    return examModel;
}
```  

테스트 데이터는 `ExamInnerModel` 이 10000 개인 리스트와 맵을 사용한다.
대략적인 결과를 비교하면 아래와 같다.  


종류|크기
---|---
Json|2.1MB
MessagePack|1.6MB(-23%)
Protobuf|1.2MB(-42%)

최대 크기인 `Json` 을 기준으로 비교하면 `MessagePack` 은 `23%` 정도의 용량 절감이 있었고, 
`Protobuf` 는 `42%` 정도의 절감으로 `Protobuf` 가 가장 효율이 좋았다. 
대부분의 경우 `Protobuf` 가 크기 절감 측면에서 가장 효율이 좋겠지만, 
절감이 되는 퍼센트는 절대적인 수치는 아니고 직렬화하는 데이터의 구성이 어떻게 돼 있냐에 따라 달라질 수 있음을 유의해야 한다.  

#### Performance Comparison
성능비교 테스트는 데이터 크기 비교와 동일한 데이터를 사용해서 직렬화/역직렬화에 대한 성능을 테스트한다. 
또한 단일 스레드 환경과 멀티 스레드 환경으로도 구분해서 진행한다.  

성능 테스트에는 `Java` 의 `Benchmark` 툴인 `JMH` 를 사용해서 진행한다. 
테스트에 앞서 `JMH` 사용을 위해 `build.gradle` 에 플러그인과 설정을 추가해 준다.  

```groovy
plugins {
  // ...
  id 'me.champeau.jmh' version '0.6.5'
}

// ...

apply plugin: 'me.champeau.jmh'

// ...

jmh {
    fork = 1
    iterations = 5
    warmupIterations = 3
}
```  

그리고 `JMH` 테스트 코드는 `scr/jmh` 디렉토리를 생성하고 그 하위에 작성하면 되는데, 
단일 스레드와 멀티 스레드 테스트 코드를 구분해서 작성해 준다.  


```java
@State(Scope.Benchmark)
@Threads(value = 1)
public class SingleThreadTest {


    public static ExamModel createExamModel() {
        List<ExamInnerModel> list = new ArrayList<>();
        Map<String, ExamInnerModel> map = new HashMap<>();
        Random random = new Random();

        for (int i = 0; i < 10000; i++) {
            ExamInnerModel examInnerModel = ExamInnerModel.builder()
                    .str(UUID.randomUUID().toString())
                    .intValue(random.nextInt())
                    .doubleValue(random.nextDouble())
                    .build();
            list.add(examInnerModel);
            map.put(String.valueOf(i), examInnerModel);
        }

        ExamModel examModel = ExamModel.builder()
                .innerModelList(list)
                .innerModelMap(map)
                .build();

        return examModel;
    }

    @State(Scope.Benchmark)
    public static class SerializeTools {
        public ObjectMapper jsonMapper = new ObjectMapper();
        public ObjectMapper msgPackMapper = new MessagePackMapper();
        public ExamModel examModel;
        public Proto.ExamModel examModelProto;
        public String jsonStr;
        public byte[] msgPackBytes;
        public byte[] protoBytes;
    }

    @Setup(Level.Trial)
    public void setUp(SerializeTools tools) throws JsonProcessingException {
        tools.examModel = createExamModel();
        tools.examModelProto = createExamModel().toProto();
        tools.jsonStr = tools.jsonMapper.writeValueAsString(tools.examModel);
        tools.msgPackBytes = tools.msgPackMapper.writeValueAsBytes(tools.examModel);
        tools.protoBytes = tools.examModelProto.toByteArray();
    }

    @Benchmark
    @BenchmarkMode(Mode.AverageTime)
    @OutputTimeUnit(TimeUnit.MICROSECONDS)
    public String jsonSerialize(SerializeTools tools) throws JsonProcessingException {
        return tools.jsonMapper.writeValueAsString(tools.examModel);
    }

    @Benchmark
    @BenchmarkMode(Mode.AverageTime)
    @OutputTimeUnit(TimeUnit.MICROSECONDS)
    public byte[] msgPackSerialize(SerializeTools tools) throws JsonProcessingException {
        return  tools.msgPackMapper.writeValueAsBytes(tools.examModel);
    }

    @Benchmark
    @BenchmarkMode(Mode.AverageTime)
    @OutputTimeUnit(TimeUnit.MICROSECONDS)
    public byte[] protobufSerialize(SerializeTools tools) {
        return tools.examModelProto.toByteArray();
    }

    @Benchmark
    @BenchmarkMode(Mode.AverageTime)
    @OutputTimeUnit(TimeUnit.MICROSECONDS)
    public ExamModel jsonDeserialize(SerializeTools tools) throws JsonProcessingException {
        return tools.jsonMapper.readValue(tools.jsonStr, ExamModel.class);
    }

    @Benchmark
    @BenchmarkMode(Mode.AverageTime)
    @OutputTimeUnit(TimeUnit.MICROSECONDS)
    public ExamModel msgPackDeserialize(SerializeTools tools) throws IOException {
        return tools.msgPackMapper.readValue(tools.msgPackBytes, ExamModel.class);
    }

    @Benchmark
    @BenchmarkMode(Mode.AverageTime)
    @OutputTimeUnit(TimeUnit.MICROSECONDS)
    public Proto.ExamModel protoDeserialize(SerializeTools tools) throws InvalidProtocolBufferException {
        return Proto.ExamModel.parseFrom(tools.protoBytes);
    }

}
```  
