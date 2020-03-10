--- 
layout: single
classes: wide
title: "[DesignPattern 개념] Mediator Pattern"
header:
  overlay_image: /img/designpattern-bg.jpg
excerpt: '복잡하게 얽혀 상호작용하는 객체를 중개인을 통해 정리하는 Mediator 패턴에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Design Pattern
tags:
  - Design Pattern
  - Mediator
use_math : true
---  

## Mediator 패턴이란
- `Mediator` 는 중재인(중개인)의 의미를 가진것 처럼, 여러개로 구성된 작업에서 객체간의 상호작용이 서로 얽혀 일어날 때 중개인이 객체에 대한 상호작용을 담당하는 것을 의미한다.
- 객체간의 상호작용을 각 각체가 수행하는 것이 아니라, 중개인이라는 하나의 매개체를 두고 중개인의 지시에 따라 객체들은 동작하게 된다.
- 중재라는 부분은 통신이라고 바꿔생각 할 수도 있다.
- 활주로에 비행기가 이륙, 착륙할 때 비행기가 서로 통신하며 순서를 정하는 것이아닌 관제소라는 중재자를 두고 일을 처리하는 것과 비슷하다.
- `GoF` 에서는 `Mediator` 패턴을 `Mediator`(중재자), `Colleague`(동료) 로 나눈다.
	- 동료는 객체간의 상호작용이 필요한 객체를 의미한다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_mediator_1.png)

- 패턴의 구성요소
	- `Mediator` : 객체간 상호작용의 중재자의 역할로, `Colleague` 와의 통신 및 역할 수행을 위한 인터페이스를 정의한다.
	- `ConcreteMediator` : `Mediator` 의 구현체로, 실제 중재자의 역할인 `Colleague` 간의 상호작용을 조정한다.
	- `Colleague` : 서로 상호작용이 필요한 객체의 역할로, 중재자의 조정을 받기위한 인터페이스를 정의한다.
	- `ConcreteColleague` : `Colleague` 의 구현체로, 실제 중재가 필요한 객체의 내용이 구현되어 있다.
- `Mediator` 와 `Colleague` 의 역할 분배
	- `Mediator` 패턴을 구성할때 중재와 관련된 부분은 `Mediator` 에 있어야 한다.
	- 중재에 대한 부분이 `Colleague` 부분에 있게 되면 전체 처리로직이 각 `Colleague` 로 분배되는 현상이 발생 할 수 있어 디버깅과 추가개발에 어려움을 줄 수 있다.
	- `Mediator` 패턴은 OOP 에서 각 처리를 분산시키는 방법과는 달리, 처리 부분 중 중재(통신)에 대한 부분은 하나의 클래스에 집중시킨다는 특징을 가지고 있다.
- 객채간의 통신경로
	- 객체간의 상호작용을 한다는 것은 n개의 객체가 있을 때, $n!$ 개의 통신경로가 필요하다.
		- `a->b`, `b->a`, `a->c`, `c->a`, `b->c`, `c->b` (a, b, c)
	- 객체의 종류나 수가 늘어날 수록 통신경로의 수는 기하급수로 늘어나기 때문에 매우복잡한 구성이 될 수 있다.
	- `Mediator` 패턴을 통해 이런 현상을 개선할 수 있다.
- `Mediator` 패턴의 재사용
	- `Mediator` 패턴에서 `Mediator` 부분은 재사용이 어렵다.
		- `Mediator` 의 경우 `Colleague` 에 대한 처리 부분까지 함께 구현되었기 때문에 관련 클래스들과 의존성이 높기 때문이다.
	- `Mediator` 패턴에서 `Colleague` 부분은 재사용 가능하다.
	
## 채팅
- 간단한 채팅관련 동작을 구현해본다.
- 유저는 채팅방에 입장하고, 채팅방에 있는 유저는 모두에게 메시지를 보낼수도 있고 귓속말처럼 한 유저에게만 메시지를 보낼 수 있다.
- 채팅방에는 여러개의 봇이 존재할 수 있고 봇은 주어진 문자열을 만족할 경우, 관련 동작을 전체 메시지로 채팅방에 발송한다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_mediator_2.png)

### ChatMessage 

```java
public class ChatMessage {
    private String fromId;
    private String toId;
    private String message;

    public ChatMessage(String fromId, String toId, String message) {
        this.fromId = fromId;
        this.toId = toId;
        this.message = message;
    }
    
    // getter, setter
}
```  

- `ChatMessage` 는 하나의 채팅메시지를 나타내는 클래스이다.
- `fromId` 는 보낸 아이디, `toId` 는 받는 아이디, `message` 는 보낸 메시지를 의미한다.
- `toId` 가 `null` 일 경우 전체 메시지를 의미한다.

### Mediator

```java
public interface Mediator {
	void addColleague(Colleague colleague);

    void broadcastMessage(ChatMessage chatMessage);

    void whisperMessage(ChatMessage chatMessage);
}
```  

- `Mediator` 는 중재자가 수행해야하는 동작을 정의한 인터페이스이다.
- `Mediator` 패턴에서 `Mediator` 역할을 수행한다.
- `addColleague()` 를 통해 상호작용에 참여할 객체(유저, 봇)을 추가할 수 있다.
- `broadcastMessage()` 로 전체 유저에게 메시지를 발송한다.
- `whisperMessage()` 로 특정 유저에게만 메시지를 발송한다.

### Colleague

```java
public abstract class Colleague {
    protected Mediator mediator;
    protected String id;

    public Colleague(String id) {
        this.id = id;
    }

    public void setMediator(Mediator mediator) {
        this.mediator = mediator;
    }

    public void sendMessage(String message) {
        this.mediator.broadcastMessage(new ChatMessage(this.id, null, message));
    }

    public String getId() {
        return id;
    }

    public abstract void receiveMessage(ChatMessage chatMessage);
}
```  

- `Colleague` 는 상호작용이 필요한 객체를 나타내면서 필요한 메소드를 정의한 추상 클래스이다.
- `Mediator` 패턴에서 `Colleague` 역할을 수행한다.
- 모든 `Colleague` 는 중재역할을 하는 `Mediator` 의 인스턴스와 아이디값을 필드로 가지고 있다.
- `sendMessage()` 메소드를 통해 상호작용에 참여한 `Colleague` 들에게 모두 메시지를 전송할 수 있다.
	- 중개인 `Mediator` 에게 상호작용에 대한 정보를 제공하는 메소드이다.
- `receiveMessage()` 는 하위 `Colleague` 구현체에서 각 객체에 맞춰 메시지 수신에 대한 처리를 구현한다.
	- 중개인 `Mediator` 가 상호작용에 대한 중재를 할떄 사용하는 메소드이다. 
	
### ColleagueUser

```java
public class ColleagueUser extends Colleague {
    private LinkedList<ChatMessage> chatMessageList;

    public ColleagueUser(String id) {
        super(id);
        this.chatMessageList = new LinkedList<>();
    }

    public void sendMessage(String id, String message) {
        this.mediator.whisperMessage(new ChatMessage(this.id, id, message));
    }

    @Override
    public void receiveMessage(ChatMessage chatMessage) {
        this.chatMessageList.addLast(chatMessage);
    }

    public LinkedList<ChatMessage> getChatMessageList() {
        return chatMessageList;
    }
}
```  

- `ColleagueUser` 는 `Colleague` 의 구현체로, 채팅에 참여한 유저를 나나태는 클래스이다.
- `Mediator` 패턴에서 `ConcreteColleague` 역할을 수행한다.
- `chatMessageList` 는 유저가 받은 메시지를 저장하는 필드다.
- `Colleague` 의 `sendMessage()` 를 오버로딩해서 특정 유저에게 메시지를 보낼 수 있다.
	- 중개인 `Mediator` 에게 상호작용에 대한 정보를 제공하는 메소드이다.
- `receiveMessage()` 를 통해 중개인에게 받은 메시지를 메시지리스트에 추가한다.

### ColleagueBot

```java
public abstract class ColleagueBot extends Colleague {
    private LinkedList<ChatMessage> chatMessageList;

    public ColleagueBot(String id) {
        super(id);
        this.chatMessageList = new LinkedList<>();
    }

    public void receiveMessage(ChatMessage chatMessage) {
        if(this.checkMessage(chatMessage)) {
            this.chatMessageList.addLast(chatMessage);
            this.doBot(chatMessage);
        }
    }

    public LinkedList<ChatMessage> getChatMessageList() {
        return this.chatMessageList;
    }

    public abstract boolean checkMessage(ChatMessage chatMessage);

    public abstract void doBot(ChatMessage chatMessage);
}
```  

- `ColleagueBot` 은 `Colleague` 의 구현체이면서, 봇에 필요한 메소드를 정의한 추상클래스이다.
- `Mediator` 패턴에서 `ConcreteColleague` 역할을 수행한다.
- `chatMessageList` 를 통해 봇이 수신한 메시지를 관리한다.
- `receiveMessage()` 를 통해 중개인으로 부터 받은 메시지가 봇 수행 메시지일 경우 봇을 수행시키고, 메시지리스트에 추가한다.
- `checkMessage()` 로 하위 클래스에서 해당 메시지가 봇 수행 메시지인지 판별한다.
- `doBot()` 으로 하위 클래스에서 각 봇이 수행하는 동작을 구현한다.

### ColleagueBotDateTime, ColleagueBotRandom

```java
public class ColleagueBotDateTime extends ColleagueBot{
    public ColleagueBotDateTime(String id) {
        super(id);
    }

    @Override
    public boolean checkMessage(ChatMessage chatMessage) {
        return chatMessage.getMessage().equals("datetime");
    }

    @Override
    public void doBot(ChatMessage chatMessage) {
        this.sendMessage(LocalDateTime.now().toString());
    }
}
```  

```java
public class ColleagueBotRandom extends ColleagueBot {
    public ColleagueBotRandom(String id) {
        super(id);
    }

    @Override
    public boolean checkMessage(ChatMessage chatMessage) {
        return chatMessage.getMessage().startsWith("random ");
    }

    @Override
    public void doBot(ChatMessage chatMessage) {
        StringTokenizer token = new StringTokenizer(chatMessage.getMessage(), " ");
        int start, end;

        if(token.countTokens() == 3) {
            try {
                token.nextToken();
                start = Integer.parseInt(token.nextToken());
                end = Integer.parseInt(token.nextToken());

                if(start <= end) {
                    Random random = new Random();
                    int tmp = end - start;
                    int rand = random.nextInt(tmp + 1) + start;
                    this.sendMessage(rand + "");
                }
            } catch(NumberFormatException e) {
                // do something
            }
        }
    }
}
```  

- `ColleagueBotDateTime`, `ColleagueBotRandom` 은 `ColleagueBot` 의 구현체로, 각 봇이 수행하는 동작을 구현한 클래스이다.
- `Mediator` 패턴에서 `ConcreteColleague` 역할을 수행한다.
- `ColleagueBotDateTime` 은 메시지가 `datetime` 이라면, 현재 날짜와 시간 정보 메시지를 전체 발송한다.
- `ColleagueBotRandom` 은 메시지가 `random` 으로 시작하고 공백을 구분자로 숫자 2개가 있으면, 두 숫자를 포함한 사이값을 랜덤으로 전체 발송한다.

### ChatRoom

```java
public class ChatRoom implements Mediator {
    private Map<String, ColleagueUser> userMap;
    private Map<String, ColleagueBot> botMap;

    public ChatRoom() {
        this.userMap = new HashMap<>();
        this.botMap = new HashMap<>();
    }

    @Override
    public void addColleague(Colleague colleague) {
        if (colleague instanceof ColleagueUser) {
            this.userMap.put(colleague.getId(), (ColleagueUser) colleague);
        } else if (colleague instanceof ColleagueBot) {
            this.botMap.put(colleague.getId(), (ColleagueBot) colleague);
        }

        colleague.setMediator(this);
    }

    @Override
    public void broadcastMessage(ChatMessage chatMessage) {
        this.broadcastUser(chatMessage);
        this.broadcastBot(chatMessage);
    }

    @Override
    public void whisperMessage(ChatMessage chatMessage) {
        String toId = chatMessage.getToId();
        Colleague toUser = this.userMap.get(toId);

        if (toUser != null) {
            toUser.receiveMessage(chatMessage);
        }
    }

    private void broadcastUser(ChatMessage chatMessage) {
        Colleague user;

        for (Map.Entry<String, ColleagueUser> entry : this.userMap.entrySet()) {
            user = entry.getValue();
            user.receiveMessage(chatMessage);
        }
    }

    private void broadcastBot(ChatMessage chatMessage) {
        Colleague bot;

        for (Map.Entry<String, ColleagueBot> entry : this.botMap.entrySet()) {
            bot = entry.getValue();
            bot.receiveMessage(chatMessage);
        }
    }
}
```  

- `ChatRoom` 은 `Mediator` 의 구현체로 채팅방을 구현한 클래스이다.
- `Mediator` 패턴에서 `ConcreteMediator` 역할을 수행한다.
- `userMap`, `botMap` 필드로 유저와 봇을 관리한다.
- `addColleague()` 메소드를 통해 `Colleague`(유저, 봇) 을 리스트에 추가하고 `Colleague` 의 `setMediator()` 메소드로 `Mediator` 인스턴스를 설정한다.
- `broadcastMessage()` 로 모든 유저와 봇 모두에게 메시지를 보낸다.
	- 모든 `Colleague` 를 중재하는 메소드
- `whistperMessage()` 로 특정 유저에게 메시지를 보낸다.
	- 특정 `Colleague` 를 중재하는 메소드
- `broadcastUser()` 는 모든 유저에게 메시지를 보낸다.
	- 다수의 특정 `Colleague` 를 중재하는 메소드
- `broadcastBot()` 은 모든 봇에게 메시지를 보낸다.
	- 다수의 특정 `Colleague` 를 중재하는 메소드

### Mediator 패턴의 처리과정
- `ColleagueUser` 와 `ColleagueBotDateTime` 가 상호작용할 때 처리과정은 아래와 같다.

![그림 1]({{site.baseurl}}/img/designpattern/2/concept_mediator_3.png)

### 테스트

```java
public class MediatorTest {
    @Test
    public void ColleagueUser_ChatMessage() {
        // given
        Mediator chatRoom = new ChatRoom();
        ColleagueUser user1 = new ColleagueUser("user1");
        ColleagueUser user2 = new ColleagueUser("user2");
        chatRoom.addColleague(user1);
        chatRoom.addColleague(user2);

        // when
        user1.sendMessage("user1 all message");
        user2.sendMessage("user2 all message");
        user1.sendMessage("user2", "user1 whisper message");

        // then
        List<ChatMessage> actualUser1 = user1.getChatMessageList();
        assertThat(actualUser1, hasSize(2));
        assertThat(actualUser1, everyItem(hasProperty("fromId", isIn(new String[]{"user1", "user2"}))));
        List<ChatMessage> actualUser2 = user2.getChatMessageList();
        assertThat(actualUser2, hasSize(3));
        assertThat(actualUser2, everyItem(hasProperty("fromId", isIn(new String[]{"user1", "user2"}))));
    }

    @Test
    public void ColleagueBotDateTime_ChatMessage() {
        // given
        Mediator chatRoom = new ChatRoom();
        ColleagueUser user1 = new ColleagueUser("user1");
        ColleagueUser user2 = new ColleagueUser("user2");
        ColleagueBotDateTime botDateTime = new ColleagueBotDateTime("BotDateTime");
        chatRoom.addColleague(user1);
        chatRoom.addColleague(user2);
        chatRoom.addColleague(botDateTime);

        // when
        user1.sendMessage("datetime");

        // then
        List<ChatMessage> actualUser1 = user1.getChatMessageList();
        assertThat(actualUser1, hasSize(2));
        assertThat(actualUser1, everyItem(hasProperty("fromId", isIn(new String[]{"user1", "BotDateTime"}))));
        List<ChatMessage> actualUser2 = user2.getChatMessageList();
        assertThat(actualUser2, hasSize(2));
        assertThat(actualUser2, everyItem(hasProperty("fromId", isIn(new String[]{"user1", "BotDateTime"}))));
    }

    @Test
    public void ColleagueBotRandom_ChatMessage() {
        // given
        Mediator chatRoom = new ChatRoom();
        ColleagueUser user1 = new ColleagueUser("user1");
        ColleagueUser user2 = new ColleagueUser("user2");
        ColleagueBotRandom botRandom = new ColleagueBotRandom("BotRandom");
        chatRoom.addColleague(user1);
        chatRoom.addColleague(user2);
        chatRoom.addColleague(botRandom);

        // when
        user1.sendMessage("random 2 100");

        // then
        List<ChatMessage> actualUser1 = user1.getChatMessageList();
        assertThat(actualUser1, hasSize(2));
        assertThat(actualUser1, everyItem(hasProperty("fromId", isIn(new String[]{"user1", "BotRandom"}))));
        List<ChatMessage> actualUser2 = user2.getChatMessageList();
        assertThat(actualUser2, hasSize(2));
        assertThat(actualUser2, everyItem(hasProperty("fromId", isIn(new String[]{"user1", "BotRandom"}))));
    }

    @Test
    public void AllColleague_ChatMessage() {
        // given
        Mediator chatRoom = new ChatRoom();
        ColleagueUser user1 = new ColleagueUser("user1");
        ColleagueUser user2 = new ColleagueUser("user2");
        ColleagueBotDateTime botDateTime = new ColleagueBotDateTime("BotDateTime");
        ColleagueBotRandom botRandom = new ColleagueBotRandom("BotRandom");
        chatRoom.addColleague(user1);
        chatRoom.addColleague(user2);
        chatRoom.addColleague(botDateTime);
        chatRoom.addColleague(botRandom);

        // when
        user1.sendMessage("datetime");
        user2.sendMessage("random 10 20");

        // then
        List<ChatMessage> actualUser1 = user1.getChatMessageList();
        assertThat(actualUser1, hasSize(4));
        assertThat(actualUser1, everyItem(hasProperty("fromId", isIn(new String[]{"user1", "user2", "BotDateTime", "BotRandom"}))));
        List<ChatMessage> actualUser2 = user2.getChatMessageList();
        assertThat(actualUser2, hasSize(4));
        assertThat(actualUser2, everyItem(hasProperty("fromId", isIn(new String[]{"user1", "user2", "BotDateTime", "BotRandom"}))));
    }
}
```  

---
## Reference

	