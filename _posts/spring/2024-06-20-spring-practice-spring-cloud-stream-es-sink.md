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
toc: true
use_math: true
---  

## Spring Cloud Stream Elasticsearch Sink
`Spring Cloud Stream Application` 중 [ElasticsearchSink](https://github.com/spring-cloud/stream-applications/blob/main/applications/sink/elasticsearch-sink/README.adoc)
은 이미 제공중인 애플리케이션이 존재한다. 
하지만 구성 환경에서 버전문제로 인해(`Elasticsearch 7.10` 에 맞는 애플리케이션 필요)
이를 다시 구현해 보는 시간을 가졌다. 
`Spring Cloud` 에서 구현해둔 것을 대부분 차용하면서 해당 코드를 분석하고 파악하는 시간도 가졌다. 
그리고 추가적으로 `Elasticsearch` 인덱스를 구성할 때 `Properties` 로 시간 포맷을 넣어 롤링이 될 수 있도록 하는 기능도 추가해 보았다.  

구현한 전체 코드는 [Elasticsearch Sink](https://github.com/windowforsun/spring-cloud-stream-es-sink)
에서 확인 할 수 있다.  

### Properties

```yaml
spring:
  cloud:
    stream:
      kafka:
        binder: localhost:9092
      function:
        bindings:
          elasticsearchConsumer-in-0: input
      bindings:
        input:
          destination: output
  elasticsearch:
    rest:
      uris: 'http://localhost:9200'
```  

`Upstream` 에서 방출되는 `out channel` 은 `ElasticsearchSink` 의 `input channel` 과 매핑되어 
`elasticsearchConsumer` 빈을 사용하는 `elasticsearchConsumer-in-0` 으로 바인딩 되어 이후 `Sink Process` 가 수행된다.   

### Properties Class
아레는 `Spring Cloud Stream` 을 사용 할때 `Properties` 로 설정할 수 있는 옵션의 내용이다.  

```java
@Data
@ConfigurationProperties("elasticsearch.sink")
public class ElasticsearchSinkProperties {
    private String index;
    private String dateTimeRollingFormat;
    private Expression id;
    private String routing;
    private long timeoutSeconds;
    private boolean async;
    private int batchSize = 1;
    private long groupTimeout = -1L;
}
```  

- `index` : `Elasticsearch` 에 문서 저장할 인덱스 이름이다. 
- `dateTimeRollingFormat` : `index` 를 생성할 때 롤링 시킬 시간 포맷을 지정하면, `index-yyyy-MM-dd` 와 같은 인덱스를 사용하게 된다. 
- `id` : `Elasticsearch` 문서 색인을 위한 아이디를 [Expression](https://docs.spring.io/spring-framework/reference/core/expressions.html)(SpEL) 형식으로 지정한다. 
- `routing` : `Elasticsearch` 문서를 저장할 때 라우팅을 지정한다. 
- `timeoutSeconds` : 문서를 저장할 때 타임아웃 시간을 지정한다. 
- `async` : 문서를 저장할 때 비동기 방식을 사용할지에 대한 설정이다. 
- `batchSize` : 문서를 저장 할 때 한번에 전송할 수를 입력한다. 
- `groupTimeout` : 문서를 `batch` 방식으로 저장 할때 `batchSize` 까지 대기할 최대 시간을 입력한다.  

`Spring Cloud Data Flow` 에서 애플리케이션을 실행하거나, 
그렇지 않더라도 `Properties` 를 사용해서 소비하는 메시지에 맞춰 `Elasticsearch` 에 문서로 저장 할 수 있다.  

### Sink Process
애플리케이션에서 `Sink Process` 의 내용은 `ElasticsearchSink` 클래스에 있다. 
메시지가 소비되면 `elasticsearchConsumerFlow` 빈이 수행되는데 `IntegrationFlow` 을 통해, 
`elasticsearchConsumer` 빈의 채널을 사용한다. 
그리고 `batchSize` 의 값을 통해 1보다 작은 경우에는 개별 메시지로 `Elasticsearch` 문서 생성 및 저장을 수행하고,  
큰 경우는 `batch` 처리를 위해 `aggregator` 를 통해 메시지 그룹화 수행 후 문서 생성 및 저장을 수행하게 된다.  

```java
@Bean
public IntegrationFlow elasticsearchConsumerFlow(AggregatingMessageHandler aggregator,
                                                 ElasticsearchSinkProperties elasticsearchSinkProperties,
                                                 MessageHandler bulkRequestHandler,
                                                 MessageHandler indexRequestHandler) {
    final IntegrationFlowBuilder builder = IntegrationFlows
            .from(Consumer.class, gateway -> gateway.beanName("elasticsearchConsumer"));

    int batchSize = elasticsearchSinkProperties.getBatchSize();

    if (batchSize > 1) {
        builder.handle(aggregator)
                .handle(bulkRequestHandler);
    } else {
        builder.handle(indexRequestHandler);
    }

    return builder.get();
}
```  

#### Single Message Sink

`batch` 방식을 사용하지 않는 경우에는 `indexRequestHandler` 를 호출하여 `createIndexRequest()` 를 사용해서 
메시지를 `Elasticsearch Document` 로 변환한다. 

```java
@Bean
public MessageHandler indexRequestHandler(RestHighLevelClient restHighLevelClient, ElasticsearchSinkProperties elasticsearchSinkProperties) {
    return message -> this.index(restHighLevelClient, createIndexRequest(message, elasticsearchSinkProperties), elasticsearchSinkProperties.isAsync());
}

private IndexRequest createIndexRequest(Message<?> message,
                                        ElasticsearchSinkProperties elasticsearchSinkProperties) {
    IndexRequest indexRequest = new IndexRequest();
    final String INDEX_ID = "INDEX_ID";
    final String INDEX_NAME = "INDEX_NAME";

    String index = (String) message.getHeaders().getOrDefault(INDEX_NAME, elasticsearchSinkProperties.getIndex());

    String dateTimeRollingFormat = elasticsearchSinkProperties.getDateTimeRollingFormat();
    if (StringUtils.isNotEmpty(dateTimeRollingFormat)) {
        try {
            String format = LocalDateTime.now().format(DateTimeFormatter.ofPattern(dateTimeRollingFormat));
            index += "-" + format;
        } catch (Exception ignore) {

        }
    }

    indexRequest.index(index);

    String id = (String) message.getHeaders().getOrDefault(INDEX_ID, StringUtils.EMPTY);

    if (elasticsearchSinkProperties.getId() != null) {
        id = elasticsearchSinkProperties.getId().getValue(message, String.class);
    }

    indexRequest.id(id);

    Object messagePayload = message.getPayload();

    if (messagePayload instanceof String) {
        indexRequest.source((String) messagePayload, XContentType.JSON);
    } else if (messagePayload instanceof Map) {
        indexRequest.source((Map<String, ?>) messagePayload, XContentType.JSON);
    } else if (messagePayload instanceof XContentBuilder) {
        indexRequest.source((XContentBuilder) messagePayload);
    }

    String routing = elasticsearchSinkProperties.getRouting();
    if (StringUtils.isNotEmpty(routing)) {
        indexRequest.routing(routing);
    }

    long timeout = elasticsearchSinkProperties.getTimeoutSeconds();
    if (timeout > 0) {
        indexRequest.timeout(TimeValue.timeValueSeconds(timeout));
    }

    log.info("createIndexRequest index : {}, payload : {}", index, messagePayload);
    return indexRequest;
}
```  

`createIndexRequest()` 에서는 `Properties` 에 지정된 인덱스 이름과 문서 아이디, 그리고 인덱스 롤링에 대한 값을 바탕으로 
`IndexRequest` 객체를 생성한다.  

그리고 최종적으로 `IndexRequest` 를 파라미터로 받는 `index()` 메소드 호출을 통해 `Elasticsearch` 에 문서를 저장한다. 
문서를 저장할 때는 `Properties` 의 `async` 값을 바탕으로 동기/비동기로 저장을 수행한다.  

```java
private void index(RestHighLevelClient restHighLevelClient,
                   IndexRequest request,
                   boolean isAsync) {
    Consumer<IndexResponse> handleResponse = response ->
            log.debug(String.format("Index operation [index=%s] succeeded: document [id=%s, version=%d] was written on shard %s.",
                    response.getIndex(), response.getId(), response.getVersion(), response.getShardId())
            );

    if (isAsync) {
        log.info("indexRequest async document desc : {}", request.getDescription());
        restHighLevelClient.indexAsync(request, RequestOptions.DEFAULT, new ActionListener<IndexResponse>() {
            @Override
            public void onResponse(IndexResponse indexResponse) {
                handleResponse.accept(indexResponse);
            }

            @Override
            public void onFailure(Exception e) {
                throw new IllegalStateException("Error occurred while indexing document: " + e.getMessage(), e);
            }
        });
    } else {
        try {
            log.info("indexRequest document desc : {}", request.getDescription());
            IndexResponse response = restHighLevelClient.index(request, RequestOptions.DEFAULT);
            handleResponse.accept(response);
        } catch (IOException e) {
            throw new IllegalStateException("Error occurred while indexing document: " + e.getMessage(), e);
        }
    }
}
```  


#### Bulk Message Sink
`batch` 방식을 사용해서 문서를 저장하는 경우 우선 `aggregator()` 로 개별 메시지를 `batchSize` 와 `groupTimeout` 시간을 바탕으로 
내부 `MessageGroupStore` 를 사용하여 `BulkRquest` 객체에 쌓아둔다.   

```java
@Bean
public AggregatingMessageHandler aggregator(MessageGroupStore messageGroupStore,
                                            ElasticsearchSinkProperties elasticsearchSinkProperties) {
    AggregatingMessageHandler handler = new AggregatingMessageHandler(
            group -> group.getMessages()
                    .stream()
                    .map(m -> createIndexRequest(m, elasticsearchSinkProperties))
                    .reduce(new BulkRequest(),
                            (bulk, indexRequest) -> {
                                bulk.add(indexRequest);
                                return bulk;
                            },
                            (bulk1, bulk2) -> {
                                bulk1.add(bulk2.requests());

                                return bulk1;
                            })
    );

    handler.setCorrelationStrategy(message -> "");
    handler.setReleaseStrategy(new MessageCountReleaseStrategy(elasticsearchSinkProperties.getBatchSize()));
    long groupTimeout = elasticsearchSinkProperties.getGroupTimeout();
    if (groupTimeout >= 0) {
        handler.setGroupTimeoutExpression(new ValueExpression<>(groupTimeout));
    }
    handler.setMessageStore(messageGroupStore);
    handler.setExpireGroupsUponCompletion(true);
    handler.setSendPartialResultOnExpiry(true);

    return handler;
}

@Bean
public MessageGroupStore messageGroupStore() {
    SimpleMessageStore messageGroups = new SimpleMessageStore();
    messageGroups.setTimeoutOnIdle(true);
    messageGroups.setCopyOnGet(false);

    return messageGroups;
}
```  

`batchSize` 를 만족하거나, `groupTimeout` 이 만료된 경우 `BulkRequest` 를 파라미터로 받는 `bulkRequestHandler` 로 
전달되고 이는 다시 `BulkRequest` 를 `Elasticsearch` 문서로 저장하는 `index()` 를 호출한다. 
해단 `index()` 도 동일하게 `async` 값을 바탕으로 동기/비동기 방식으로 문서 저장방식을 결정한다.  

```java
@Bean
public MessageHandler bulkRequestHandler(RestHighLevelClient restHighLevelClient,
                                         ElasticsearchSinkProperties elasticsearchSinkProperties) {
    return message -> this.index(restHighLevelClient, (BulkRequest) message.getPayload(), elasticsearchSinkProperties.isAsync());
}

private void index(RestHighLevelClient restHighLevelClient,
	BulkRequest request,
	boolean isAsync) {
	Consumer<BulkResponse> handleResponse = responses -> {
		if (log.isDebugEnabled() || responses.hasFailures()) {
			for (BulkItemResponse itemResponse : responses) {
				if (itemResponse.isFailed()) {
					log.error(String.format("Index operation [i=%d, id=%s, index=%s] failed: %s",
						itemResponse.getItemId(), itemResponse.getId(), itemResponse.getIndex(), itemResponse.getFailureMessage())
					);
				} else {
					DocWriteResponse r = itemResponse.getResponse();
					log.debug(String.format("Index operation [i=%d, id=%s, index=%s] succeeded: document [id=%s, version=%d] was written on shard %s.",
						itemResponse.getItemId(), itemResponse.getId(), itemResponse.getIndex(), r.getId(), r.getVersion(), r.getShardId())
					);
				}
			}
		}

		if (responses.hasFailures()) {
			throw new IllegalStateException("Bulk indexing operation completed with failures: " + responses.buildFailureMessage());
		}
	};

	if (isAsync) {
		log.info("bulkRequest async document desc : {}", request.getDescription());
		restHighLevelClient.bulkAsync(request, RequestOptions.DEFAULT, new ActionListener<BulkResponse>() {
			@Override
			public void onResponse(BulkResponse bulkItemResponses) {
				handleResponse.accept(bulkItemResponses);
			}

			@Override
			public void onFailure(Exception e) {
				throw new IllegalStateException("Error occurred while performing bulk index operation: " + e.getMessage(), e);
			}
		});
	} else {
		try {
			log.info("bulkRequest document desc : {}", request.getDescription());
			BulkResponse bulkResponse = restHighLevelClient.bulk(request, RequestOptions.DEFAULT);
			handleResponse.accept(bulkResponse);
		} catch (Exception e) {
			throw new IllegalStateException("Error occurred while performing bulk index operation: " + e.getMessage(), e);
		}
	}
}
```  

### Test
테스트는 `Spring Cloud Stream` 을 간단하게 테스트 할 수 있는 `TestChannelBinderConfiguration` 의 `InputDestionation` 을 사용해서
채널로 부터 메시지를 소비한다. 
그리고 로컬의 `Docker` 환경을 사용하는 `TestContainers` 를 통해 `Elasticsearch` 환경을 구성해 `Elasticsearch` 로 문서가 저장될 수 있도록 구성했다.  

테스트는 크게 아래오 같이 구성해 진행했다.  

- 배치를 사용하지 않고, 인덱스 롤링을 사용하지 않는 경우
- 배치를 사용하고, 인덱스 롤링을 사용하는 경우
- 배치를 사용하고, 인덱스 롤링을 사용하지 않는 경우
- 비동기 저장, 배치를 사용하고, 인덱스 롤링을 사용하는 경우 

아래는 테스트 클래스 구성의 예시이다.  

```java
@ExtendWith(SpringExtension.class)
@Import(TestChannelBinderConfiguration.class)
@Testcontainers
@SpringBootTest(
        properties = {
                "elasticsearch.sink.index=test",
                "elasticsearch.sink.batch-size=3",
                "elasticsearch.sink.group-timeout=2",
                "elasticsearch.sink.date-time-rolling-format=yyyy-MM-dd",
                "elasticsearch.sink.async=true"
        }
)
@ActiveProfiles("test")
public class AsyncBatchTimeBasedIndexTest {
	private static String INDEX = "test";
	@Autowired
	private InputDestination inputDestination;
	@Autowired
	private RestHighLevelClient restHighLevelClient;
	@Autowired
	private ElasticsearchSinkProperties properties;
	@Container
	public static ElasticsearchContainer container = new ElasticsearchContainer(
		"docker.elastic.co/elasticsearch/elasticsearch:7.10.0");

	@DynamicPropertySource
	static void elasticsearchProperties(DynamicPropertyRegistry registry) {
		registry.add("spring.elasticsearch.rest.uris", () -> container.getHttpHostAddress());
	}

	@BeforeEach
	public void setUp() throws IOException {
		INDEX +=
			"-" + LocalDateTime.now().format(DateTimeFormatter.ofPattern(this.properties.getDateTimeRollingFormat()));
	}

	@Test
	public void json_string_message_ok() throws Exception {
        // ...
	}
}
```  


---  
## Reference
[Elasticsearch Sink](https://github.com/spring-cloud/stream-applications/blob/main/applications/sink/elasticsearch-sink/README.adoc)  


