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
- `ChronoUnit` : 년, 월, 일, 시, 분, 초 등의 한 가지 단위로 길이를 표현하기 위한 클래스이다.
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
        assertThat(today.toString(), is("2019-11-25"));

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
        ZonedDateTime nowKr = ZonedDateTime.now();
        assertThat(nowKr.toString(), is(allOf(
                containsString("2019-11-25"),
                containsString("Asia/Seoul")
        )));

        ZonedDateTime nowLa = ZonedDateTime.now(ZoneId.of("America/Los_Angeles"));
        assertThat(nowLa.toString(), is(allOf(
                containsString("2019-11-25"),
                containsString("America/Los_Angeles")
        )));

        ZonedDateTime somePlaceDateTime = ZonedDateTime.of(
                1992, 10, 1,
                11,11,11,0,
                ZoneId.of("Asia/Seoul"));
        assertThat(somePlaceDateTime.toString(), is(allOf(
                containsString("1992-10-01"),
                containsString("11:11:11"),
                containsString("Asia/Seoul")
        )));

        LocalDate date = LocalDate.of(1992, 10, 1);
        LocalTime time = LocalTime.of(11, 11, 11);
        LocalDateTime dateTime = LocalDateTime.of(date, time);
        ZonedDateTime zonedDateTime = ZonedDateTime.of(dateTime, ZoneId.of("Asia/Seoul"));
        assertThat(zonedDateTime.toLocalDate().toString(), is("1992-10-01"));
        assertThat(zonedDateTime.toLocalTime().toString(), is("11:11:11"));

        // UTC 기준 한국과 시간차는 +9시간
        ZoneOffset seoulZoneOffset = ZoneOffset.of("+09:00");
        ZonedDateTime nowKr2 = ZonedDateTime.now().withZoneSameInstant(ZoneId.of("Asia/Seoul"));
        assertThat(nowKr2.format(DateTimeFormatter.ISO_LOCAL_DATE_TIME), is(ZonedDateTime.now(seoulZoneOffset).format(DateTimeFormatter.ISO_LOCAL_DATE_TIME)));

        ZonedDateTime seoul = Year.of(1992).atMonth(10).atDay(1).atTime(11, 11, 11).atZone(ZoneId.of("Asia/Seoul"));
        ZonedDateTime utc = seoul.withZoneSameInstant(ZoneOffset.UTC);
        assertThat(utc.format(DateTimeFormatter.ISO_LOCAL_DATE_TIME), is(seoul.minusHours(9).format(DateTimeFormatter.ISO_LOCAL_DATE_TIME)));

        ZonedDateTime shanghai = seoul.withZoneSameInstant(ZoneId.of("Asia/Shanghai"));
        assertThat(shanghai.format(DateTimeFormatter.ISO_LOCAL_DATE_TIME), is(seoul.minusHours(1).format(DateTimeFormatter.ISO_LOCAL_DATE_TIME)));

        String isoDateTimePattern = somePlaceDateTime.format(DateTimeFormatter.ISO_DATE_TIME);
        assertThat(isoDateTimePattern, is("1992-10-01T11:11:11+09:00[Asia/Seoul]"));
        String customPattern = somePlaceDateTime.format(DateTimeFormatter.ofPattern("yyyy/MM/dd HH:mm:ss z"));
        assertThat(customPattern, is("1992/10/01 11:11:11 KST"));
        String krPattern = somePlaceDateTime.format(DateTimeFormatter.ofLocalizedDateTime(FormatStyle.FULL));
        assertThat(krPattern, is("1992년 10월 1일 목요일 오전 11시 11분 11초 KST"));
        String usPattern = somePlaceDateTime.format(DateTimeFormatter.ofLocalizedDateTime(FormatStyle.FULL).withLocale(Locale.US));
        assertThat(usPattern, is("Thursday, October 1, 1992 11:11:11 AM KST"));

        ZonedDateTime isoDateTime = ZonedDateTime.parse(isoDateTimePattern);
        assertThat(isoDateTime, is(somePlaceDateTime));
        ZonedDateTime customDateTime = ZonedDateTime.parse(customPattern, DateTimeFormatter.ofPattern("yyyy/MM/dd HH:mm:ss z"));
        assertThat(customDateTime, is(somePlaceDateTime));
        ZonedDateTime krDateTime = ZonedDateTime.parse(krPattern, DateTimeFormatter.ofLocalizedDateTime(FormatStyle.FULL));
        assertThat(krDateTime, is(somePlaceDateTime));
        ZonedDateTime usDateTime = ZonedDateTime.parse(usPattern, DateTimeFormatter.ofLocalizedDateTime(FormatStyle.FULL).withLocale(Locale.US));
        assertThat(usDateTime, is(somePlaceDateTime));
    }


    @Test
    public void Duration_Test() {
        LocalTime start = LocalTime.of(11, 11, 11, 0);
        LocalTime end = LocalTime.of(11, 12, 11, 11);
        Duration duration = Duration.between(start, end);
        assertThat(duration.getSeconds(), is(60l));
        assertThat(duration.getNano(), is(11));

        Duration oneMinutes = Duration.ofMinutes(1);
        assertThat(oneMinutes.getSeconds(), is(60l));
        Duration oneHours = Duration.ofHours(1);
        assertThat(oneHours.getSeconds(), is(3600l));
        Duration oneDays = Duration.ofDays(1);
        assertThat(oneDays.getSeconds(), is(3600l * 24));

        Duration parseDuration = Duration.parse("PT10H36M50.008S");
        assertThat(parseDuration.getSeconds(), is(38210l));
        assertThat(parseDuration.getNano(), is(8000000));
    }

    @Test
    public void Period_Test() {
        LocalDate start = LocalDate.of(1992, 10, 1);
        LocalDate end1 = LocalDate.of(2019, 1, 1);
        Period period1 = Period.between(start, end1);
        assertThat(period1.getYears(), is(26));
        assertThat(period1.getMonths(), is(3));
        assertThat(period1.getDays(), is(0));

        LocalDate end2 = LocalDate.of(2019, 11, 25);
        Period period2 = Period.between(start, end2);
        assertThat(period2.getYears(), is(27));
        assertThat(period2.getMonths(), is(1));
        assertThat(period2.getDays(), is(24));

        Period custom = Period.of(11, 11, 11);
        assertThat(custom.getYears(), is(11));
        assertThat(custom.getMonths(), is(11));
        assertThat(custom.getDays(), is(11));

        Period parse = Period.parse("P1Y2M3D");
        assertThat(parse.getYears(), is(1));
        assertThat(parse.getMonths(), is(2));
        assertThat(parse.getDays(), is(3));
    }

    @Test
    public void ChronoUnit_Test() {
        LocalDate startDate = LocalDate.of(1992, 10, 1);
        LocalDate endDate = LocalDate.of(2019, 1, 1);
        long years = ChronoUnit.YEARS.between(startDate, endDate);
        long months = ChronoUnit.MONTHS.between(startDate, endDate);
        long weeks = ChronoUnit.WEEKS.between(startDate, endDate);
        long days = ChronoUnit.DAYS.between(startDate, endDate);
        assertThat(years, is(26l));
        assertThat(months, is(315l));
        assertThat(weeks, is(1369l));
        assertThat(days, is(9588l));


        LocalTime startTime = LocalTime.of(11, 11, 11, 0);
        LocalTime endTime = LocalTime.of(12, 13, 14, 15);
        long hours = ChronoUnit.HOURS.between(startTime, endTime);
        long minutes = ChronoUnit.MINUTES.between(startTime, endTime);
        long seconds = ChronoUnit.SECONDS.between(startTime, endTime);
        assertThat(hours, is(1l));
        assertThat(minutes, is(62l));
        assertThat(seconds, is(3723l));
    }
}
```  
	
---
## Reference
[]()  
