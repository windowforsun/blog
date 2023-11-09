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
  - Reactive Stream
  - Reactor
  - Backpressure
  - Flux
toc: true 
use_math: true
---  

## Java Generic to Protobuf
`Protobuf` 는 직렬화 대상이 되는 데이터 크기를 많이 줄 일 수 있고, 
직렬화 성능 또한 좋다. 
하지만 이런 이점들을 얻기 위해서는 직렬화 대상이 되는 데이터에 대해 스키마를 별도로 지정해야 한다. 
또한 스키마에서 데이터 필드는 명시적인 데이터 타입을 가져야 한다는 점이 있다.  

위와 같은 이유로 `Protobuf` 에서 런타임에 타입이 결정되는 `Java Generic` 에 대해서는 공식적으로 
지원하는 기능은 없다. 
하지만 `Protobuf` 에서 제공하는 몇가지 기능을 함께 사용하는 트릭을 사용하면, 
어느정도 `Java Generic` 을 `Protobuf` 스키마로 표현해서 사용 할 수 있다.  

### Oneof
[Oneof](https://protobuf.dev/programming-guides/proto3/#oneof)
는 `Protobuf` 메시지 특정 필드에 여러 필드나 타입이 올 수 있을 때, 
`oneof` 집합 중 하나의 필드를 사용 할 수 있도록 해준다.  


```protobuf
message Sample {
  oneof test_oneof {
    string str = 1;
    int32 num = 2;
    OtherMessage other_message = 3;
  }
}
```  

위 예시처럼 `test_oneof` 라는 `oneof` 집합을 구성하는 3개의 필드중 하나의 필드만 선택적으로 사용 될 수 있고, 
사용되지 않은 나머지 필드는 모두 지워진다.  

### Wrapper Message
`Wrapper Message` 란 표현 그대로 `Protobuf` 의 `Message` 를 동적 타입 표현을 목적으로 
한번 더 `Message` 로 감싼 형태를 의미한다. 
그 예시는 아래와 같다.  

```protobuf
message MyMessage {
  SampleWrapper sample = 1;
}

message SampleWrapper {
  oneof sample_oneof {
    SampleA sample_a = 1;
    SampleB sample_b = 2;
  }
}
```  

`MyMessage` 메시지 스키마를 기준으로 `sample` 이름을 가진 필드는 
필요에 따라 런타임에 `SampleA`, `SampleB` 2가지 타입이 올 수 있는 필드로 사용 할 수 있다.  

### Example
간단한 `Java` 코드를 통해 `Java Generic` 을 `Protobuf` 스키마로 구성하는 예시에 대해 살펴본다. 
`build.gradle` 은 [여기]()
를 참고해서 구성 할 수 있다. 
아래는 `Java` 도메인 클래스의 예시이다.  

```java
public class ExamModel<T extends ExamInnerModel> {
    private List<T> innerModelList;
    private Map<String, T> innerModelMap;
}

public interface ExamInnerModel {
}

public class ExamInnerModelA implements ExamInnerModel {
  private String strA;

}

public class ExamInnerModelB implements ExamInnerModel {
  private int numberB;

}
```  

위 도메인 클래스 구조를 `Oneof` 와 `Wrapper Message` 를 사용해서 `Protobuf` 스키마로 표현하면 아래와 같다.  

```protobuf
syntax = "proto3";
package com.windowforsun.generic.proto;

option java_package = "com.windowforsun.generic.proto";
option java_outer_classname = "Proto";

message ExamModel {
  repeated ExamInnerModel inner_model_list = 1;
  map<string, ExamInnerModel> inner_model_map = 2;
}

message ExamInnerModel {
  oneof value_oneof {
    ExamInnerModelA ExamInnerModelA = 1;
    ExamInnerModelB ExamInnerModelB = 2;
  }
}

message ExamInnerModelA {
  string str_a = 1;
}

message ExamInnerModelB {
  int32 number_b = 1;
}
```  

기존 도메인 클래스에서 `Protobuf` 클래스로 변환하는 코드를 추가하면 아래와 같이 작성 할 수 있다.  

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ExamModel<T extends ExamInnerModel> {
    private List<T> innerModelList;
    private Map<String, T> innerModelMap;

    public Proto.ExamModel toProto() {

        return Proto.ExamModel.newBuilder()
                .addAllInnerModelList(this.innerModelList
                        .stream()
                        .map(t -> Proto.ExamInnerModel.newBuilder()
                                .setField(Proto.ExamInnerModel
                                        .getDescriptor()
                                        .findFieldByName(t.getClass().getSimpleName()), t.toProto())
                                .build())
                        .collect(Collectors.toList()))
                .putAllInnerModelMap(this.innerModelMap
                        .entrySet()
                        .stream()
                        .collect(Collectors.<Map.Entry<String, T>, String, Proto.ExamInnerModel>toMap(
                                Map.Entry::getKey,
                                entry -> Proto.ExamInnerModel.newBuilder()
                                        .setField(Proto.ExamInnerModel
                                                        .getDescriptor()
                                                        .findFieldByName(entry.getValue().getClass().getSimpleName()),
                                                entry.getValue().toProto())
                                        .build()
                        )))
                .build();
    }
}

public interface ExamInnerModel {
  MessageLite toProto();
}

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ExamInnerModelA implements ExamInnerModel {
  private String strA;

  @Override
  public MessageLite toProto () {
    return Proto.ExamInnerModelA.newBuilder()
            .setStrA(this.strA)
            .build();
  }
}

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ExamInnerModelB implements ExamInnerModel {
  private int numberB;

  @Override
  public MessageLite toProto() {
    return Proto.ExamInnerModelB.newBuilder()
            .setNumberB(this.numberB)
            .build();
  }
}
```  

동작을 검증하는 테스트 코드를 작성하면 아래와 같다.  

```java
public class GenericTest {
  @Test
  public void generic_to_json() throws JsonProcessingException {
    List<ExamInnerModelA> innerModelAList = new ArrayList<>();
    List<ExamInnerModelB> innerModelBList = new ArrayList<>();
    Map<String, ExamInnerModelA> innerModelAMap = new HashMap<>();
    Map<String, ExamInnerModelB> innerModelBMap = new HashMap<>();

    for (int i = 0; i < 2; i++) {
      ExamInnerModelA a = ExamInnerModelA.builder()
              .strA("a" + i)
              .build();
      innerModelAList.add(a);
      innerModelAMap.put(String.valueOf(i), a);

      ExamInnerModelB b = ExamInnerModelB.builder()
              .numberB(i)
              .build();
      innerModelBList.add(b);
      innerModelBMap.put(String.valueOf(i), b);
    }

    // create ExamModel Java Generic
    ExamModel<ExamInnerModelA> aExamModel = ExamModel.<ExamInnerModelA>builder()
            .innerModelList(innerModelAList)
            .innerModelMap(innerModelAMap)
            .build();
    ExamModel<ExamInnerModelB> bExamModel = ExamModel.<ExamInnerModelB>builder()
            .innerModelList(innerModelBList)
            .innerModelMap(innerModelBMap)
            .build();

    // encode json
    ObjectMapper objectMapper = new ObjectMapper();
    String aJson = objectMapper.writeValueAsString(aExamModel);
    String bJson = objectMapper.writeValueAsString(bExamModel);

    // decode json
    ExamModel<ExamInnerModelA> decodeAExamModel = objectMapper.readValue(aJson, new TypeReference<ExamModel<ExamInnerModelA>>() {
    });
    ExamModel<ExamInnerModelB> decodeBExamModel = objectMapper.readValue(bJson, new TypeReference<ExamModel<ExamInnerModelB>>() {
    });

    Assertions.assertEquals(aExamModel.getInnerModelList().get(0).getStrA(), decodeAExamModel.getInnerModelList().get(0).getStrA());
    Assertions.assertEquals(aExamModel.getInnerModelList().get(1).getStrA(), decodeAExamModel.getInnerModelList().get(1).getStrA());
    Assertions.assertEquals(aExamModel.getInnerModelMap().get("0").getStrA(), decodeAExamModel.getInnerModelMap().get("0").getStrA());
    Assertions.assertEquals(aExamModel.getInnerModelMap().get("1").getStrA(), decodeAExamModel.getInnerModelMap().get("1").getStrA());
    Assertions.assertEquals(bExamModel.getInnerModelList().get(0).getNumberB(), decodeBExamModel.getInnerModelList().get(0).getNumberB());
    Assertions.assertEquals(bExamModel.getInnerModelList().get(1).getNumberB(), decodeBExamModel.getInnerModelList().get(1).getNumberB());
    Assertions.assertEquals(bExamModel.getInnerModelMap().get("0").getNumberB(), decodeBExamModel.getInnerModelMap().get("0").getNumberB());
    Assertions.assertEquals(bExamModel.getInnerModelMap().get("1").getNumberB(), decodeBExamModel.getInnerModelMap().get("1").getNumberB());
  }

  @Test
  public void generic_to_protobuf() throws InvalidProtocolBufferException {
    List<ExamInnerModelA> innerModelAList = new ArrayList<>();
    List<ExamInnerModelB> innerModelBList = new ArrayList<>();
    Map<String, ExamInnerModelA> innerModelAMap = new HashMap<>();
    Map<String, ExamInnerModelB> innerModelBMap = new HashMap<>();

    for (int i = 0; i < 2; i++) {
      ExamInnerModelA a = ExamInnerModelA.builder()
              .strA("a" + i)
              .build();
      innerModelAList.add(a);
      innerModelAMap.put(String.valueOf(i), a);

      ExamInnerModelB b = ExamInnerModelB.builder()
              .numberB(i)
              .build();
      innerModelBList.add(b);
      innerModelBMap.put(String.valueOf(i), b);
    }

    // create ExamModel Java Generic
    ExamModel<ExamInnerModelA> aExamModel = ExamModel.<ExamInnerModelA>builder()
            .innerModelList(innerModelAList)
            .innerModelMap(innerModelAMap)
            .build();
    ExamModel<ExamInnerModelB> bExamModel = ExamModel.<ExamInnerModelB>builder()
            .innerModelList(innerModelBList)
            .innerModelMap(innerModelBMap)
            .build();

    // generate protobuf class
    Proto.ExamModel protoAExamModel = aExamModel.toProto();
    Proto.ExamModel protoBExamModel = bExamModel.toProto();

    // encode protobuf
    byte[] aBytes = protoAExamModel.toByteArray();
    byte[] bBytes = protoBExamModel.toByteArray();

    // decode protobuf
    Proto.ExamModel decodeAExamModel = Proto.ExamModel.parseFrom(aBytes);
    Proto.ExamModel decodeBExamModel = Proto.ExamModel.parseFrom(bBytes);

    Assertions.assertEquals(aExamModel.getInnerModelList().get(0).getStrA(), decodeAExamModel.getInnerModelListList().get(0).getExamInnerModelA().getStrA());
    Assertions.assertEquals(aExamModel.getInnerModelList().get(1).getStrA(), decodeAExamModel.getInnerModelListList().get(1).getExamInnerModelA().getStrA());
    Assertions.assertEquals(aExamModel.getInnerModelMap().get("0").getStrA(), decodeAExamModel.getInnerModelMapMap().get("0").getExamInnerModelA().getStrA());
    Assertions.assertEquals(aExamModel.getInnerModelMap().get("1").getStrA(), decodeAExamModel.getInnerModelMapMap().get("1").getExamInnerModelA().getStrA());
    Assertions.assertEquals(bExamModel.getInnerModelList().get(0).getNumberB(), decodeBExamModel.getInnerModelListList().get(0).getExamInnerModelB().getNumberB());
    Assertions.assertEquals(bExamModel.getInnerModelList().get(1).getNumberB(), decodeBExamModel.getInnerModelListList().get(1).getExamInnerModelB().getNumberB());
    Assertions.assertEquals(bExamModel.getInnerModelMap().get("0").getNumberB(), decodeBExamModel.getInnerModelMapMap().get("0").getExamInnerModelB().getNumberB());
    Assertions.assertEquals(bExamModel.getInnerModelMap().get("1").getNumberB(), decodeBExamModel.getInnerModelMapMap().get("1").getExamInnerModelB().getNumberB());
  }
}
```  

`Json` 을 사용하는 직렬화와는 약간 사용 방식에 차이점은 있지만, 
`Java Generic` 형태에 대해서 `Protobuf` 를 사용해서 직렬화/역직렬화가 가능함을 알 수 있다.  

하지만 위 방식은 단점이 있는데, 
`Wrapper Message` 를 사용한 방식이기 때문에 데이터의 전체 깊이가 `Wrapper` 의 수 만큼 늘어 난다는 점이 있다. 
`Generic` 당 하나의 `Wrapper Message` 를 사용한다면 메시지에 표현에 사용된 `Generic` 의 수 만큼 깊이가 깊어 질 수 있다는 것을 주의해야 한다.  


---
## Reference
[]()  

