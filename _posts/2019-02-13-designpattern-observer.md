--- 
layout: post
title: "디자인 패턴(Design Pattern) 옵저버 패턴(Observer Pattern)"
subtitle: '옵저버 패턴이 무엇이고, 어떠한 특징을 가지는지'
author: "window_for_sun"
header-style: text
tags:
    - 디자인 패턴
    - Design Pattern
    - Observer Pattern
---  
[참고](https://lee-changsun.github.io/2019/02/08/designpattern-intro/)  

### 옵저버 패턴이란
- 한 객체의 상태 변화에 따라 다른 객체의 상태도 연동되도록 일대다 객체 의존 관계를 구성 하는 패턴
    - 데이터의 변경이 발생했을 경우 상대 클래스나 객체에 의존하지 않으면서 데이터 변경을 통보 하고자 할 때 유용하다.
        - 새로운 파일이 추가되거나 기존 파일이 삭제되었을 때 탐색기는 다른 탐색기에게 즉시 변경을 통보해야 한다.
        - 차량 연료량 클래스는 연료량이 부족한 경우 연료량에 관심을 가지는 구체적인 클래스(연료량 부족 경고 클래스, 주행 가능 거리 출력 클래스)에 직접 의존하지 않는 방식으로 연료량 변화를 통보해야 한다.
        - 행위 패턴 중 하나
    - ![옵저버 패턴 예시1](/../img/designpattern-observer-ex-1-classdiagram.png)
        - Observer
            - 데이터의 변경을 통보 받는 인터페이스
            - Subject에서는 Observer 인터페이스의 update 메서드를 호출 함으로써 ConcreteSubject의 데이터 변경을 ConcreteObserver에게 통보한다.
        - Subject
            - ConcreteObserver 객체를 관리하는 요소
            - Observer 인터페이스를 참조해서 ConcreteObserver를 관리하므로 ConcreteObserver의 변화에 독립적일 수 있다.
        - ConcreteSubject
            - 변경 관리 대상이 되는 데이터가 있는 클래스(통보하는 클래스)
            - 데이터 변경을 위한 메서드인 setState()가 있다.
            - setState() 메서드에서는 자신의 데이터인 subjectState를 변경하고 Subject의 notifyObservers 메서드를 호출해서 ConcreteObserver에 변경을 통보한다.
        - ConcreteObserver
            - ConcreteSubject의 변경을 통보 받는 클래스
            - Observer 인터페이스의 update 메서드를 구현함으로써 변경을 통보받는다.
            - 변경된 데이터는 ConcreteSubject의 getState메서드를 호출함으로써 변경을 조회한다.
            
### 예시
#### 여러 가지 방식으로 성적 출력하기
- ![옵저버 패턴 성적 1](/../img/designpattern-observer-score-1-classdiagram.png)
- 입력된 성적값을 출력하는 프로그램

```java
// 입력된 점수를 저장하는 클래스 (1. 출력 형태 : 목록)
public class ScoreRecord {
    // 점수 저장
    private List<Integer> scores = new ArrayList<Integer>();
    
    // 목록 형태로 점수를 출력하는 클래스
    private DataSheetView dataSheetView;
    
    public void setDataSheetView(DataSheetView dataSheetView) {
        this.dataSheetView = dataSheetView;
    }
    
    public void addScore(int score){
        this.scores.add(score);
        this.dataSheetView.update();
    }
    
    public List<Integer> getScoreRecord() {
        return this.scores;
    }
}

// 1. 출력 형태 : 목록 형태로 출력하는 클래스
public class DataSheetView {
    private ScoreRecord scoreRecord;
    private int viewCount;
    
    public DataSheetView(ScoreRecord scoreRecord, int viewCount){
        this.scoreRecord = scoreRecord;
        this.viewCount = viewCount;
    }
    
    // 점수의 변경을 통보받음
    public void update() {
        // 점수 조회
        List<Integer> record = this.scoreRecord.getScoreRecord();
        // 조회된 점수를 viewCount 만큼만 출력함
        this.displayScores(record, viewCount);
    }
    
    private void displayScores(List<Integer> record, int viewCount){
        System.out.println("List of " + viewCount + " entries : ");
        int size = record.size();
        for(int i = 0; i < viewcount && i < size; i++){ 
            System.out.println(record.get(i) + " ");
        }
        System.out.println();
    }
}
```  

```java
public class Client {
    public static void main(String[] args){
        ScoreRecord scoreRecord = new ScoreRecord();
        
        DataSheetView dataSheetView = new DataSheetView(scoreRecord, 3);
        scoreRecord.setDataSheetView(dataSheetView);
        
        for(int index = 1; index <= 5; index++){
            int score = index * 10;
            System.out.println("Adding " + score);
            
            // 10 20 30 40 50을 추가함, 추가할 때마다 최대 3개의 점수만 출력함
            scoreRecord.addScore(score);
        }
    }
}
```   

- ScoreRecord 클래스의 addScore 메서드가 호출되면 ScoreRecord 클래스는 자신의 필드인 scores 객체에 점수를 추가한다.
- DataSheetView 클래스는 ScoreRecord클래스의 getScoreRecord 메서드를 호출해 출력할 점수를 구한다.
    - DataSheetView 클래스의 update메서드에서는 구한 점수 중에서 명시된 개수(viewCount)만큼의 점수만 출력한다.
    
#### 문제점
1. 성적을 다른 형태로 출력하는 경우
    - 성적을 목록으로 출력하지 않고 성적의 최소/최대 값만 출력하려면 ?
    
    ```java
    // 입력된 점수를 저장하는 클래스 (2. 출력형태 : 최대, 최소값)
    public class ScoreRecord {
        private List<Integer> scores = new ArrayList<Integer>();
        private MinMaxView minMaxView;
        
        // MinMaxView를 설정함
        public void setMinMaxView(MinMaxView minMaxView) {
            this.minMaxView = minMaxView;
        }
        
        // 새로운 점수를 추가하면 ㅊ출력하는 것에 변화를 통보(update())하여 출력하는 부분
        public void addScore(int score){
            // scores 목록에 주어진 점수를 추가함
            this.scores.add(score);
            // MinMaxView에게 scores가 변경됨을 통보함
            this.minMaxView.update();
        }
        
        // 출력하는 부분에서 변화된 내용을 얻어감
        public List<Integer> getScoreRecord() {
            return this.scores;
        }
    }
    
    // 2. 출력형태 : 최소/최대 값만을 출력하는 형태의 클래스
    public class MinMaxView {
        private ScoreRecord scoreRecord;
        
        // getScoreRecord()를 호출하기 위해 ScoreRecord 객체를 인자로 받음
        public MinMaxView(ScoreRecord scoreRecord){
            this.scoreRecord = scoreRecord;
        }
        
        // 점수의 변경을 통보받음
        public void update() {
            // 점수 조회
            List<Integer> record = this.scoreRecord.getScoreRecord();
            // 점수 출력
            this.displayScores(record);
        }
        
        // 최소/최대 값을 출력함
        private void displayScores(List<Integer> record) {
            int min = Collections.min(record, null);
            int max = Collections.max(record, null);
            
            System.out.println("Min : " + min + ", Max : " + max);
        }
    }
    ```  
    
    - 점수 변경에 대한 통보 대상 클래스가 다른 대상 클래스(DataSheetView -> MinMaxView)로 바뀌면 기존 코드(ScoreRecord 클래스)의 내용을 수정해야 하므로 OCP에 위배된다.
1. 동시 혹은 순차적으로 성적을 출력하는 경우
    - 성적이 입력되었을 때 최대 3개 목록, 최대 5개 목록, 최소/최대 값을 동시에 출력하려면 ??
    - 처음에는 목록을 출력하고 나중에는 최소/최대 값을 출력하려면 ?
    
    ```java
    // 입력된 점수를 저장하는 클래스 (3. 출력형태 : 2개 출력 형태를 가질때)
    public class ScoreRecord {
        private List<Integer> scores = new ArrayList<Integer>();
        private DataSheetView dataSheetView;
        private MinMaxView minMaxView;
        
        public void setDataSheetView(DataSheetView dataSheetView) {
            this.dataSheetView = dataSheetView;
        }
        
        public void setMinMaxView(MinMaxView minMaxView){
            this.minMaxView = minMaxView;
        }
        
        // 새로운 점수를 추가하면 출력하는 것에 변화를 통보(update()) 하여 출력하는 부분 갱신
        public void addScore(int score){
            this.scores.add(score);
            this.dataSheetView.update();
            this.minMaxView.update();
        }
        
        public List<Integer> getScoreRecord(){
            return this.scores;
        }
    }
    ```  
    
    ```java
    public class Client {
        public static void main(String[] args){
            ScoreRecord scoreRecord = new ScoreRecord();
            // 3개까지의 점수만 출력함
            DataSheetView dataSheetView = new DataSheetView(scoreRecord, 3);
            // 최소/최대 값만 출력함
            MinMaxView minMaxView = new MinMaxView(scoreRecord);
            
            // 각 통보 대상 클래스 저장
            scoreRecord.setDataSheetView(dataSheetView);
            scoreRecord.setMinMaxView(minMaxView);
            
            // 10 20 30 40 50 을 추가
            for(int index = 1; index <= 5; index++){
                int score = index * 10l;
                System.out.println("Adding : " + score);
            }
            
            // 추가 할때마다 최대 3개의 점수 목록과 최소/최대 값 출력
            scoreRecord.addScore(score);
        }
    }
    ```  
    
    - 이 경우에도 점수 변경에 대한 통보 대상 클래스가 다른 대상 클래스(DataSheetView -> MinMaxView)로 바뀌면 기존 코드(ScoreRecord 클래스)의 내용을 수정해야 하므로 OCP에 위배된다.
    - 성적 변경을 새로운 클래스에 통보할 때마다 ScoreRecord 클래스의 코드를 수정해야 하므로 재사용이 어렵다.
- ![옵저버 패턴 문제 1](/../img/designpattern-observer-problem-1-classdiagram.png)

#### 해결 방법
문제를 해결하기 위해서는 공통 기능을 상위 클래스 및 인터페이스로 일반화 하고 이를 활용하여 통보하는 클래스(ScoreRecord 클래스)를 구현해야 한다.
- ScoreRecord클래스에서 변화되는 부분을 식별하고 이를 일반화 시켜야 한다.
- 이를 통해 성적 통보 대상이 변경되더라도 ScoreRecord 클래스를 그대로 재사용할 수 있다.
- ScoreRecord 클래스에서 하는 작업
    - 통보 대상인 객체를 참조하는 것을 관리(추가/제거) -> Subject 클래스로 일반화
    - addScore 메서드 : 각 통보 대상인 객체의 update 메서드를 호출 -> Observer 인터페이스로 일반화
- ![옵저버 패턴 점수 해결 1](/../img/designpattern-observer-solution-1-classdiagram.png)
1. ScoreRecord 클래스의 addScore(상태 변경) 메서드 호출
    1. 자산의 성적 값 저장
    1. 상태가 변경 될 때마다 Subject 클래스의 notifyObservers 메서드 호출
1. Subject 클래스의 notifyObservers 메서드 호출
    1. Observer 인터페이스를 통해 성적 변경을 통보
    1. DataSheetView, MinMaxView 클래스의 update 메서드 호출

- Observer 인터페이스

```java
// 추상화 된 통보 대상
public interface Observer {
    // 데이터가 변경을 통보했을 때 처리하는 메서드
    public void update();
}
```  

- Subject 클래스

```java
// 추상화 된 변경 관심 대상 데이터
// 데이터에 공통적으로 들어가야하는 메서드들 -> 일반화
public abstract class Subject {
    // 추상화된 통보 대상 목록 (출력 형태에 대한 Observer)
    private List<Observer> observers = new ArrayList<Observer>();
    
    // 통보 대상(Observer) 추가
    public void attach(Observer observer) {
        this.observers.add(observer);
    }
    
    // 통보 대상(Observer) 제거
    public void detach(Observer observer) {
        this.observers.remove(observer);
    }
    
    // 각 통보 대상(Observer)에 변경을 통보
    public void notifyObservers() {
        for(Observer o : this.observers) {
            o.update();
        }
    }
}
```  

- ScoreRecord 클래스

```java
// 구체적인 변경 감시 대상 데이터
// 출력형태 2개를 가질 때
public class ScoreRecord extends Subject {
    private List<Integer> scores = new ArrayList<Integer>();
    
    // 새로운 점수 추가(상태 변경)
    public void addScore(int score) {
        // scores 목록에 주어진 점수 추가
        this.scores.add(score);
        // socres 가 변경됨을 각 통보 대상(Observer)에게 통보함
        super.notifyObservers();
    }
    
    public List<Integer> getScoreRecord() {
        return this.scores;
    }
}
```  

- DataSheetView 클래스

```java
// 통보 대상 클래스(update 메서드 구현)
// 출력형태 : 목록 형태로 출력
public class DataSheetView implements Observer {
    // 동일
}
```  

- MinMaxView 클래스

```java
// 통보 대상 클래스(update 메서드 구현)
// 출력형태 : 최소/최대값 출력
public class MinMaxView implements Observer {
    // 동일
}
```  

- 클라이언트에서의 사용

```java
public class Client {
    public static void main(String[] args){
        ScoreRecord scoreRecord = new ScoreRecord();
        
        // 3개 까지의 점수만 출력함
        DataSheetView dataSheetView = new DataSheetView(scoreRecord, 3);
        // 최소/최대값만 출력함
        MinMaxView minMaxView = new MinMaxView(scoreRecord, 3);
        
        // 각 통보 대상 클래스를 Observer로 추가
        scoreRecord.attach(dataSheetView);
        scoreRecord.attach(minMaxView);
        
        // 10 20 30 40 50을 추가
        for(int index = 0; index <= 5; index++){
            int score = index * 10;
            System.out.println("Adding " + score);
            
            // 차가할 때마다 최대 3개의 점수 목록과 최대/최소값이 출력됨
            scoreRecord.addScore(score);
        }
    }
}

```  

- Observer
    - 추상화된 통보 대상
- DataSheetView, MinMaxView (ConcreteObserver)
    - Observer를 implements 함으로써 구체적인 통보 대상이 됨
- Subject
    - 성적 변경에 관심이 있는 대상 객체들을 관리
- ScoreRecord (ConcreteSubject)
    - Subject를 extends 함으로써 구체적인 통보 대상을 직접 참조하지 않아도 됨
- 이렇게 Observer 패턴을 이용하면 ScoreRecord 클래스의 코드를 변경하지 않고도 새로운 관심 클래스 및 객체를 추가/제거하는 것이 가능해진다.


---
 
## Reference
[[Design Pattern] 옵저버 패턴이란](https://gmlwjd9405.github.io/2018/07/08/observer-pattern.html)
