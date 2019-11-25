--- 
layout: single
classes: wide
title: "[Java 개념] 날짜, 시간관련 처리"
header:
  overlay_image: /img/java-bg.jpg
excerpt: 'Java 1.8 이후 날짜와 시간 처리에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Java
tags:
    - Practice
    - Java
---  

## Java 날짜 시간 처리
- Java 1.8 이전에는 날짜와 시간처리를 위해 `Date` 이나 `Calendar` 를 사용했었다.
- `Calendar`, `Date` 클래스의 문제점은 [네이버 D2](https://d2.naver.com/helloworld/645609)에 자세하게 기술돼있다.
	- 불변 객체가 아니다. (set 을 통해 변경가능하다)
	- 상수 필드가 너무 많이 사용된다.
	- 혼란스러운 월 값 (0월 부터 시작된다)
	- 일관성 없는 요일(월요일이 0, 1 로 시작)
	- Date, Calendar 두개의 객체를 번갈아가며 사용해야 한다.
- 위와 같은 문제점들로 인해 Java 1.8 이후 부터 새로운 날짜, 시간 관련 클래스가 추가되었다.

## 새로운 날짜, 시간관련 클래스
### 날짜(Temporal)
- `Instant` : Machine time 으로 UTC 기준시 1970년 1월 1일 부터 시간을 사용할 수 있는 클래스로, Milliseconds, Nanoseconds 까지 지원하며 출력은 ISO 포맷을 사용한다.
- `LocalDateTime` : 년, 월, 일, 시, 분, 초를 표현하는 클래스이다.
- `LocalDate` : 년, 월, 일과 같은 날짜만 표현하는 클래스이다.
- `LocalTime` : 시, 분, 초와 같은 시간만 표현하는 클래스이다.
- `ZonedDateTime` : `LocalDateTime` 과 달리 타임존(시차) 개념을 가지고 있는 클래스이다.

### 기간(TemporalAmount)
- `Period` : `LocalDate` 사용해서 두 날짜 사이(년, 월, 일)의 기간을 표현한다.
- `Duration` : `Instant` 를 사용해서 두 시간 사이(일, 시, 분, 초)의 기간을 표현한다.

### 기타
- `ChronoUnit` : 년, 월, 일, 시, 분, 초 등의 한 가지 단위를 표현하기 위한 클래스이다.
- `DayOfWeek` : 요일을 표현하는 Enum 이다.
- `Month` : 월을 표현하는 Enum 이다.

## 사용 및 테스트

```java
public class JavaTimeClassTest {
    @Test
    public void Instant_Test() {
        // Instant 는 UTC 기준시 ISO 포맷
        // Instant 의 초기값 1970-01-01T00:00:00Z
        assertThat(Instant.EPOCH.toString(), is(containsString("1970-01-01")));
        assertThat(Instant.ofEpochSecond(0).toString(), is(containsString("1970-01-01")));

        // 초기값 + 2,000,000,000 초는 2033-05-18T03:33:20Z
        System.out.println(Instant.ofEpochSecond(2_000_000_000).toString());
        assertThat(Instant.ofEpochSecond(2_000_000_000).toString(), is(containsString("2033-05-18")));

        // 초기값 - 2,000,000,000 초는 1906-08-16T20:26:40Z
        System.out.println(Instant.ofEpochSecond(-2_000_000_000).toString());
        assertThat(Instant.ofEpochSecond(-2_000_000_000).toString(), is(containsString("1906-08-16")));

        // 현재 시간
        Instant current = Instant.now();
        assertThat(current.toString(), is(containsString("2019-11-25")));
        assertThat(current.getEpochSecond(), is(lessThanOrEqualTo(System.currentTimeMillis() / 1000)));
        assertThat(current.toEpochMilli(), is(lessThanOrEqualTo(System.currentTimeMillis())));

        // 초를 이용한 연산
        assertThat(current.plusSeconds(60).getEpochSecond(), is(current.getEpochSecond() + 60));
        assertThat(current.minusSeconds(60).getEpochSecond(), is(current.getEpochSecond() - 60));

        // 초를 이용한 비교
        Instant after = current.plusSeconds(60);
        Instant before = current.minusSeconds(60);
        assertThat(after.isAfter(current), is(true));
        assertThat(before.isBefore(current), is(true));
    }

    @Test
    public void LocalDate_Test() {
        LocalDate today = LocalDate.now();
        assertThat(today.toString(), is("2019-11-26"));

        LocalDate parseDay = LocalDate.parse("2019-10-01");
        assertThat(parseDay.toString(), is("2019-10-01"));

        LocalDate someDay = LocalDate.of(1992, 10, 1);
        assertThat(someDay.toString(), is("1992-10-01"));

        LocalDate after1Day = someDay.plusDays(1);
        assertThat(after1Day.toString(), is("1992-10-02"));
    }

    @Test
    public void LocalTime_Test() {
        // 현재 시간
        LocalTime currentTime = LocalTime.now();
        assertThat(currentTime.toString(), is(containsString("01:")));

        // 현재 중국시간 한국시간 -1
        LocalTime currentCnTime = LocalTime.now(ZoneId.of("Asia/Shanghai"));
        LocalTime expectedCnTime = currentTime.minusHours(1);
        assertThat(currentCnTime, is(greaterThanOrEqualTo(expectedCnTime)));

        LocalTime parseTime = LocalTime.parse("11:11:11");
        assertThat(parseTime.toString(), is("11:11:11"));

        LocalTime someTime = LocalTime.of(1, 1, 1);
        assertThat(someTime.toString(), is("01:01:01"));

        LocalTime after1Hours = someTime.plusHours(1);
        assertThat(after1Hours.toString(), is("02:01:01"));
    }

    @Test
    public void LocalDateTime_Test() {
        LocalDateTime now = LocalDateTime.now();
        assertThat(now.toString(), is(containsString("2019-11-26T01:")));

        LocalDateTime combinationNow = LocalDateTime.of(LocalDate.now(), LocalTime.now());
        assertThat(now.toString(), is(containsString("2019-11-26T01:")));

        LocalDateTime parseDateTime = LocalDateTime.parse("1111-11-11T11:11:11.111");
        assertThat(parseDateTime.toString(), is("1111-11-11T11:11:11.111"));

        LocalDateTime someDateTime = LocalDateTime.of(1992, 10, 1, 1, 1, 1);
        assertThat(someDateTime.toString(), is("1992-10-01T01:01:01"));

        LocalDateTime someDateTime2 = Year.of(1992).atMonth(10).atDay(1).atTime(1, 1, 1);
        assertThat(someDateTime2.toString(), is("1992-10-01T01:01:01"));

        LocalDateTime after1Month = someDateTime.plusMonths(1);
        assertThat(after1Month.toString(), is("1992-11-01T01:01:01"));
    }

    @Test
    public void LocalDateTime_LocalDate_LocalTime_Util_Test() {
        LocalDate someDay = Year.of(1992).atMonth(10).atDay(1);
        assertThat(someDay.toString(), is("1992-10-01"));
        assertThat(someDay.getYear(), is(1992));
        assertThat(someDay.getMonth(), is(Month.OCTOBER));
        assertThat(someDay.getDayOfMonth(), is(1));
        assertThat(someDay.getDayOfWeek(), is(DayOfWeek.THURSDAY));
        assertThat(someDay.isLeapYear(), is(true));
        assertThat(someDay.plusYears(1).getYear(), is(1993));
        assertThat(someDay.plusMonths(1).getMonth(), is(Month.NOVEMBER));
        assertThat(someDay.plusDays(1).getDayOfMonth(), is(2));

        LocalDate beforeSomeDate = someDay.minusDays(1);
        assertThat(someDay.isAfter(beforeSomeDate), is(true));
        assertThat(someDay.isBefore(beforeSomeDate), is(false));

        LocalDateTime someDateTime = someDay.atTime(11, 11, 11, 0);
        assertThat(someDateTime.toString(), is("1992-10-01T11:11:11"));
        assertThat(someDateTime.getHour(), is(11));
        assertThat(someDateTime.getMinute(), is(11));
        assertThat(someDateTime.getSecond(), is(11));
        assertThat(someDateTime.getNano(), is(0));
    }

    @Test
    public void ZonedDateTime_Test() {

    }

    @Test
    public void Period_Test() {

    }

    @Test
    public void Duration_Test() {
        
    }
}
```  
	
---
## Reference
[]()  
