![그림 1]({{site.baseurl}}/img/spring/practice-springbootspringprofile-1.png)

[Two Pointers]({{site.baseurl}}{% link _posts/2019-06-03-algorithm-concept-twopointers.md %})

{% raw %}

{% endraw %}







<details><summary>출력 로그</summary>
<div markdown="1">

```
04:27:56.752 [main] INFO com.windowforsun.reactor.scheduler.SchedulersTest - onSubscribe(FluxCreate.BufferAsyncSink)
04:27:56.754 [main] INFO com.windowforsun.reactor.scheduler.SchedulersTest - request(unbounded)
04:27:57.760 [main] INFO com.windowforsun.reactor.scheduler.SchedulersTest - onNext(1)
04:27:57.760 [main] INFO com.windowforsun.reactor.scheduler.SchedulersTest - subscribe_1 : 1
04:27:58.771 [main] INFO com.windowforsun.reactor.scheduler.SchedulersTest - onNext(2)
04:27:58.771 [main] INFO com.windowforsun.reactor.scheduler.SchedulersTest - subscribe_1 : 2
04:27:59.781 [main] INFO com.windowforsun.reactor.scheduler.SchedulersTest - onNext(3)
04:27:59.781 [main] INFO com.windowforsun.reactor.scheduler.SchedulersTest - subscribe_1 : 3
04:27:59.781 [main] INFO com.windowforsun.reactor.scheduler.SchedulersTest - onComplete()
04:27:59.782 [main] INFO com.windowforsun.reactor.scheduler.SchedulersTest - onSubscribe(FluxCreate.BufferAsyncSink)
04:27:59.782 [main] INFO com.windowforsun.reactor.scheduler.SchedulersTest - request(unbounded)
04:28:00.796 [main] INFO com.windowforsun.reactor.scheduler.SchedulersTest - onNext(1)
04:28:00.796 [main] INFO com.windowforsun.reactor.scheduler.SchedulersTest - subscribe_2 : 1
04:28:01.810 [main] INFO com.windowforsun.reactor.scheduler.SchedulersTest - onNext(2)
04:28:01.810 [main] INFO com.windowforsun.reactor.scheduler.SchedulersTest - subscribe_2 : 2
04:28:02.824 [main] INFO com.windowforsun.reactor.scheduler.SchedulersTest - onNext(3)
04:28:02.824 [main] INFO com.windowforsun.reactor.scheduler.SchedulersTest - subscribe_2 : 3
04:28:02.824 [main] INFO com.windowforsun.reactor.scheduler.SchedulersTest - onComplete()
```  

</div>
</details>

```sql
 create table team_4_rel_iden
 (
	 id   bigint not null,
	 name varchar(255),
	 primary key (id)
 );
create table member_4_rel_iden
(
	id   bigint not null,
	name varchar(255),
	primary key (id)
);
create table team_member_4_rel_iden
(
	timestamp timestamp,
	member_id bigint not null,
	team_id   bigint not null,
	primary key (member_id, team_id)
);

alter table team_member_4_rel_iden
	add constraint FKnxwvehycu9ays0qnwdabioyoh foreign key (member_id) references member_4_rel_iden;
alter table team_member_4_rel_iden
	add constraint FKd2amsbqfbb96mbbdiiug3i2ep foreign key (team_id) references team_4_rel_iden;





```  




