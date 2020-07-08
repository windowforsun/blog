--- 
layout: single
classes: wide
title: "[Linux 실습] 타임존(TimeZone), 시간 변경하기"
header:
  overlay_image: /img/linux-bg-2.jpg
excerpt: 'Linux 에서 Timezone 을 변경하는 방법에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Linux
tags:
  - Linux
  - TimeZone
  - Practice
---  

## 시간 변경하기
- `date -s "년:월-일 시:분:초"` 와 같은 형식으로 입력한다.

```
[root@windowforsun windowforsun]# date
Thu Aug 15 18:32:12 KST 2019
[root@windowforsun windowforsun]# date -s "2002-06-25 00:00:00"
Tue Jun 25 00:00:00 KST 2002
[root@windowforsun windowforsun]# date
Tue Jun 25 00:00:01 KST 2002
```  

## 타임존(Timezone) 변경하기
- hwclock 는 하드웨어 시간을 뜻한다.

```
[root@windowforsun windowforsun]# ln -sf /usr/share/zoneinfo/Asia/Seoul /etc/localtime
[root@windowforsun windowforsun]# rdate -s time.nist.gov
[root@windowforsun windowforsun]# hwclock --systohc
[root@windowforsun windowforsun]# date
Thu Aug 15 18:35:30 KST 2019
[root@windowforsun windowforsun]# hwclock
Thu 15 Aug 2019 06:35:46 PM KST  -0.403393 seconds
```  

---
## Reference
[리눅스 시간 변경](https://zetawiki.com/wiki/%EB%A6%AC%EB%88%85%EC%8A%A4_%EC%8B%9C%EA%B0%84_%EB%B3%80%EA%B2%BD)  