--- 
layout: post
title: "디자인 패턴(Design Pattern) 커맨드 패턴(Command Pattern)"
subtitle: '커맨드 패턴이 무엇이고, 어떠한 특징을 가지는지'
author: "window_for_sun"
header-style: text
tags:
    - 디자인 패턴
    - Design Pattern
    - Command Pattern
---  
[참고](https://lee-changsun.github.io/2019/02/08/designpattern-intro/)  

### 커맨드 패턴이란
- 실행될 기능을 캡슐화함으로써 주어진 여러 기능을 실행할 수 있는 재사용성이 높은 클래스를 설계하는 패턴
    - 이벤트가 발생했을 때 실행될 기능이 다양하면서도 변경이 필요한 경우에 이벤트를 발생시키는 클새르를 변경하지 않고 재사용하고자 할 때 유용하다.
    - 행위 패턴의 하나
    - ![커맨드 패턴 예시1](/../img/designpattern-command-ex-classdiagram.png)
    - 실행될 기능을 캡슐화함으로써 기능이 실행 요구하는 호출자(Invoker) 클래스와 실제 기능을 실행하는 수신자(Receiver) 클래스 사이의 의존성을 제거한다.
    - 역할이 수행하는작업
        - Command
            - 실행될 기능에 대한 인터페이스
            - 실행될 기능을 execute 메서드로 선언함
        - ConcreteCommand
            - 실제로 실행되는 기능을 구현
            - Command라는 인터페이스를 구현함
        - Invoker
            - 기능의 실행을 요청하는 호출자 클래스
        - Receiver
            - ConcreteCommand에서 execute 메서드를 구현할 때 필요한 클래스
            - ConcreteCommand의 기능을 실행하기 위해 사용하는 수신자 클래스
            
### 예시
#### 만능 버튼 만들기
- ![커맨드 패턴 버튼 예시 1](/../img/designpattern-command-button-1-classdiagram.png)
- 버튼이 눌리면 램프의 불이 켜지는 프로그램  

```java
public class Lamp {
    public void turnOn() {
        System.out.println("Lamp On");
    }
}

public class Button {
    private Lamp lamp;
    
    public Button(Lamp lamp) {
        this.lamp = lamp;
    }
    
    public void pressed() {
        this.lamp.turnOn();
    }
}
```  

```java
public class Client {
    public static void main(String[] args){
        Lamp lamp = new Lamp();
        
        Button lampButton = new Button(lamp);
        lampButton.pressed();
    }
}
```

- Button클래스의 생성자를 이용해 불을 켤 Lamp 객체를 전달한다.
- Button클래스의 press()가 호출되면 생성자를 통해 전달받은 Lamp객체의 trunOn()을 호출해 불을 켠다.

#### 문제점
1. 버튼을 눌렀을 때 다른 기능을 실행하는 경우
    - 버튼을 눌렀을 때 알림이 시작되게 하려면 ?  
    
    ```java
    public class Alarm {
        public void start() {
            System.out.println("Alarming");
        }
    }
    
    public class Button {
        private Alarm alarm;
        
        public Button(Alarm alarm){
            this.alarm = alarm;
        }
        
        public void pressed() {
            this.alarm.start();
        }
    }
    ```  
    
    ```java
    public class Client {
        public static void main(String[] args){
            Alarm alarm = new Alarm();
            
            Button alarmButton = new Button(alarm);
            alarmButton.pressed();
        }
    }
    ```  
    
    - 새로운 기능으로 변경하려고 기존(Button 클래스)의 내용을 수정해야 하므로 OCP에 위배된다.
    - Button클래스의 pressed() 전체를 변경해야 한다.

1. 버튼을 누르는 동작에 따라 다른 기능을 실행하는 경우
    - 버튼을 처음 눌렀을 때는 램프를 켜고, 두 번째 눌렀을 때는 알림을 동작하게 하려면 ?  
    
    ```java
    enum Mode {
        LAMP, ALARM
    }
    
    public class Button {
        private Lamp lamp;
        private Alarm alarm;
        private Mode mode;

        public Button(Lamp lamp, Alarm alarm) {
            this.lamp = lamp;
            this.alarm = alarm;
        }
        
        public void setMode(Mode mode) {
            this.mode = mode;
        }
        
        public void pressed() {
            switch(this.mode){
                case LAMP:
                    this.lamp.turnOn();
                    break;
                case ALARM:
                    this.alarm.start();
                    break;
            }
        }
    }
    ```  
    
    - 필요한 기능을 새로 추가할 때마다 Button 클래스의 코드를 수정해야 하므로 재사용하기 어렵다.
    
#### 해결 방법
문제를 해결하기 위해서는 구체적인 기능을 직접 구현하는 대신 실행될 기능을 캡슐화해야 한다.
- Button 클래스의 pressed 메서드에서 구체적인 기능(램프 켜기, 알람 동작 등)을 직접 구현하는 대신 버튼을 놀렀을 때 실행될 기능을 Button 클래스 외부에서 제공받아 캡슐화해 pressed메서드에서 호출한다.
- 이를 통해 Button클래스 코드를 수정하지 않고도 그대로 사용할 수 있다.
- ![커맨트 패턴 해결 1](/../img/designpattern-command-button-solution-1-classdiagram.png)
    - Button클래스는 미리 약속된 Command 인터페이스의 execute 메서드를 호출한다.
        - 램프를 켜는 경우에는 lamp.turnOn() 메서드를 호출하고
        - 알람이 동작하는 경우에는 alarm.start() 메서드를 호출하도록 pressed 메서드를 수정한다.
    - LampOnCommand 클래스에서는 Command 인터페이스의 execute 메서드를 구현해 Lamp 클래스의 turnOn() 메서드(램프 켜는 기능)를 호출한다.
    - 마찬가지로 AlarmStartCommand 클래스는 Command 인터페이스의 execute 메서드를 구현해 Alarm 클래스의 start() 메서드(알람이 울리는 기능)를 호출한다.
- Command 인터페이스  

```java
public interface Command {
    public void execute();
}
```  

- Button 클래스  

```java
public class Button {
    private Command command;
    
    // 버튼을 눌렀을 때 필요한 기능을 인자로 받는다.
    public Button(Command command) {
        this.setCommand(command);
    }
    
    public void setCommand(Command command) {
        this.command = command;
    }
    
    // 버튼이 눌리면 주어진 Command.execute() 메서드를 호출한다.
    public void pressed() {
        this.command.execute();
    }
}
```  

- Lamp, LampOnCommand 클래스  

```java
public class Lamp {
    public void turnOn() {
        System.out.println("Lamp On");
    }
}

// 램프를 켜는 LampOnCommand
public class LampOnCommand implements Command {
    private Lamp lamp;
    
    public LampOnCommand(Lamp lamp) {
        this.lamp = lamp;
    }
    
    @Override
    public void execute() {
        this.lamp.turnOn();
    }
}
```  

- Alarm, AlarmStartCommand 클래스  

```java
public class Alarm {
    public void start() {
        System.out.println("Alarming");
    }
}

// 알람을 울리는 AlarmStartCommand
public class AlarmStartCommand implements Command {
    private Alarm alarm;
    
    public AlarmStartCommand(Alarm alarm) {
        this.alarm = alarm;
    }
    
    @Override
    public void execute() {
        this.alarm.start();
    }
}
```  

- 클라이언트에서의 사용  

```java
public class Client {
    public static void main(String[] args){
        Lamp lamp = new Lamp();
        Command lampOnCommand = new LampOnCommand(lamp);
        Alarm alarm = new Alarm();
        Command alarmStartCommand = new AlarmStartCommand(alarm);
        
        // 램프 켜는 Command 설정
        Button button1 = new Button(lampOnCommand);
        // 램프 켜기
        button1.pressed();
        
        
        // 알람 울리는 Command 설정
        Button button2 = new Button(alarmStartCommand);
        // 알람 시작
        button2.pressed();
        // 다시 램프 켜는 Command 설정
        button2.setCommand(lampOnCommand);
        // 램프 켜기
        button2.pressed();
    }
}
```  

- Command 인터페이스를 구현하는 LampOnCommand, AlarmStartCommand 객체를 Button 객체에 설정한다.
- Button 클래스의 pressed 메서드에서 Command 인터페이스의 execute 메서드를 호출한다.
- 버튼을 눌렀을 때 필요한 임의의 기능은 Command 인터페이스를 구현한 클래스의 객체를 Button 객체에 설정해서 실행할 수 있다.
- 이렇게 Command 패턴을 이용하면 Button 클래스의 코드를 변경하지 않으면서 다양한 동작을 구현할 수 있게 된다.

### 정리
![커맨드 패턴 버튼 정리](/../img/designpattern-command-button-conclusion-classdiagram.png)


---
 
## Reference
[[Design Pattern] 커맨드 패턴이란](https://gmlwjd9405.github.io/2018/07/07/command-pattern.html)
