--- 
layout: single
classes: wide
title: "[Algorithm 실습] 가중치 기반 랜덤 선출"
header:
  overlay_image: /img/algorithm-bg.jpg
excerpt: '가중치를 기반으로 랜덤 선출하는 알고리즘에 대해 알아보자'
author: "window_for_sun"
header-style: text
categories :
  - Algorithm
tags:
  - Algorithm
use_math : true
---  

## 가중치 기반 랜덤 선출
- 게임 기획에서 `특정 가중치에 의해 랜덤으로 선출` 을 하는 방식은 자주 등장한다.
- 아이템 뽑기, 아이템 강화, 상점 리스트 선출, 캐릭터 능력치 수치 등 다양한 기획에서 등장한다.
- 관련 개발을 하며 초기버전과 이를 개선한 방법에 대해 소개한다.
- 여기서 가중치라는 의미는 확률과도 밀접한 관련이 있다. 가중치는 전체 가중치를 가진 풀(Pool) 중에서 각 요소들이 얼마의 확률을 가지고 있는지를 의미한다.
- 가중치에 대한 확률이란, 각 요소의 가중치의 값이 바뀌거나 가중치의 요소가 추가/삭제가 될때마다 유동적으로 확률 값이 바뀌게 된다.

### 예시 기획
- 1주일 정산마다 랭킹에 따른 가중치를 차등으로 부여한다.
- 랭킹에 대한 가중치를 기반으로 1등급 보상, 2등급 보상, 3등급 보상 순으로 지급한다.
- 한번 보상에 선정된 유저는 제외된다.

### 예시 데이터
- 랜덤 선출에 사용할 가중치는 아래와 같고, `entryMap` 이라고 한다.

	key|0|1|2|3
	---|---|---|---|---
	weight|10|20|30|40 
	
- 4개의 가중치를 가지고 선출을 하게 될때, 각 가중치의 값들이 가질 확률을 아래와 같다.

	key|0|1|2|3
	---|---|---|---|---
	probability|10%|20%|30%|40%


## 기본 소스코드 구성
### WeightEntry

```java
@Getter
@Setter
@AllArgsConstructor
@Builder
public class WeightEntry<K> {
    protected K key;
    protected long weight;
}
```  

- `WeightEntry` 는 가중치에 대한 정보를 표현하는 클래스이다.
- `key` 필드는 가중치를 고유하게 지칭할 수 있는 값으로, Generic 타입으로 선언돼 있다.
- `weight` 는 가중치의 실제 값을 의미하는 값이다.

### WeightSelector

```java
public abstract class WeightSelector<K, V extends WeightEntry<K>> {
    protected Map<K, V> entryMap;
    protected long totalWeight;
    protected Random random;

    public WeightSelector() {
        this.entryMap = new HashMap<>();
        this.totalWeight = 0;
        this.random = new Random();
    }

    public void addEntry(V entry) {
        this.totalWeight += entry.getWeight();
        this.entryMap.put(entry.getKey(), entry);
    }

    public void removeEntry(K key) {
        this.totalWeight -= this.entryMap.remove(key).getWeight();
    }

    public long getRandomWeight() {
        return Math.abs(this.random.nextLong()) % this.totalWeight;
    }

    protected abstract List<K> getSelectedKeyList(int needCount, boolean isDuplicated);

    public boolean checkProcess(int needCount, boolean isDuplicated) {
        boolean result = true;

        if(!isDuplicated && this.entryMap.size() < needCount) {
            result = false;
        }

        return result;
    }

    public final List<K> processSelectKey(int needCount, boolean isDuplicated) throws Exception {
        if(!this.checkProcess(needCount, isDuplicated)) {
            throw new Exception("Fail to check process");
        }
        return this.getSelectedKeyList(needCount, isDuplicated);
    }
}
```  

- `WeightSelector` 는 가중치 정보(`WeightEntry`) 를 바탕으로 가중치를 기반으로 랜덤 선출하기 위해 필요한 메소드가 정의된 추상 클래스이다.
- 2개의 Generic 타입을 사용한다.
	- `K` : `WeightEntry` 에서 사용하는 가중치의 고유 키에 대한 타입
	- `V` : 가중치 정보를 나타내는 클래스의 타입으로 `WeightEntry` 의 하위 클래스여야 한다.
- 선출시에 사용하는 전체 가중치 정보는 `Map` 형식으로 `entryMap` 필드에 저장한다.
- `totalWeight` 는 전체 가중치의 총 합을 나태내는 필드이다.
- `Random` 클래스를 통해 랜덤 값을 사용한다.
- `addEntry` 메소드는 `WeightEntry` 를 인자값으로 받아 `entryMap` 에 추가하는데, 추가할때 `totalWeight` 에 인자값의 가중치를 더해준다.
- `removeEntry` 메소드는 인자값으로 가중치 정보의 키값을 받아 `entryMap` 에서 삭제하고, `totalWeight` 에서 가중치 만금 차감해준다.
- `getRandomWeight()` 
- `processSelectKey()` 메소드는 인자 값으로 선출할 개수와, 중복 허용 여부를 받아 `checkProcess()` 메소드와 `getSelectedKeyList()` 메소드를 사용해서 선출 된 키를 `List` 로 반환한다.
	- `getSelectedKeyList()` 메소드는 하위 클래스에서 구현한 내용대로 키를 선출해 `List` 형식으로 반환한다.
	- `checkProcess()` 메소드는 선출 전 유효성 검사를 수행한다.


## 초기 방식
- 초기 개발버전에 사용했던 방식이고, 구현이 간단한 방법이다.
- 예시로 3개의 키를 선출해 본다.
- 아래와 같은 가중치 데이터가 있을 때, 모든 가중치의 합인 `totalWeight` 를 구한다.

	key|0|1|2|3
	---|---|---|---|---
	weight|10|20|30|40 
	
	- `totalWeight` 는 100 이 된다.
- 선출 할때 마다 0 ~ 100(`totlaWeight`) 중 랜덤 값을 하나 뽑고, 뽑힌 값을 `randomWeight` 라고 한다.
- `randomWeight` 의 값이 50이라고 했을 때 `entryMap` 을 차례대로 순회하며 `randomWeight` 보다 첫 번째로 큰 `weight` 의 `key` 를 구한다.
	- 이때 순회하며 `randomWeight` 보다 `weight` 값이 크지 않다면 `randomWeight` 에 `key` 에 해당하는 `weight` 값을 빼준다.

	key|0|1|2|3
	---|---|---|---|---
	weight|10|20|30|40 
	randomWeight|50|40|20|.

- `key` 가 2인 `weight` 값 30 이 `randomWeight` 값 20보다 크기 때문에, 첫번째로 선출된 가중치의 `key` 는 2가 된다.
- 선출된 가중치를 지우면 아래와 같이 되고, `totalWeight` 는 70이 된다.

	key|0|1|3
	---|---|---|---
	weight|10|20|40 
	
- 다시 0 ~ 70(`totalWeight`) 중 랜덤 값을 뽑아 `randomWeight` 의 값은 0 일때, 다시 `entryMap` 을 순회하며 `randomWeight` 보다 크지만 가장 작은 값을 찾는다.

	key|0|1|3
	---|---|---|---
	weight|10|20|40 
	randomWeight|0|.|.
	
- `key` 0 의 `weight` 인 10 이 0(`randomWeight`) 보다 크기 때문에, 두번째 선출된 가중치의 `key` 는 0이 된다.
- 마지막으로 가중치를 지우면 아래와 같이 되고, `totalWeight` 는 60이 된다.

	key|1|3
	---|---|---
	weight|20|40 
	
- 마지막으로 0 ~ 60(`totalWeight`) 중 랜덤 값을 뽑아 `randomWeight` 의 값은 30 일때, 다시 `entryMap` 을 순회하며 `randomWeight` 보다 크지만 가장 작은 값을 찾는다.

	key|1|3
	---|---|---
	weight|20|40 
	randomWeight|30|10
	
- `key` 3 의 `weight` 인 40 이 10(`randomWeight`) 보다 크기 때문에, 마지막으로 선출된 가중치의 `key` 는 3이 되서 최종적으로 뽑힌 가중치의 키는 `2, 0, 3`(뽑힌 순서대로) 가 된다.

### 소스코드

![그림 1]({{site.baseurl}}/img/algorithm/practice_randomweightselector_1.png)

#### ProtoWeightSelector

```java
public class ProtoWeightSelector<K, V extends WeightEntry<K>> extends WeightSelector<K, V> {
    @Override
    protected List<K> getSelectedKeyList(int needCount, boolean isDuplicated) {
        List<K> selectedKeyList = new LinkedList<>();
        long randomWeight, entryWeight;
        K selectedKey;
        // 가중치 풀
        Set<Map.Entry<K, V>> entrySet = this.entryMap.entrySet();

        // 선출할 수만큼 반복
        for(int i = 0; i < needCount; i++) {
            // i 번째 선출시 사용할 랜덤 가중치 값
            randomWeight = this.getRandomWeight();
            selectedKey = null;

            // 가중치 풀을 순회하며 랜덤 가중치 값 보다 큰 값 찾기
            for(Map.Entry<K, V> entry : entrySet) {
                entryWeight = entry.getValue().getWeight();
                if(randomWeight < entryWeight) {
                    // 랜덤 가중치 값보다 크면, 선택된 키로 지정
                    selectedKey = entry.getKey();
                    break;
                } else {
                    // 랜덤 가중치 값보다 크지 않다면, i 번째 가중치 값만큼 차감
                    randomWeight -= entryWeight;
                }
            }

            if(selectedKey == null) {
                throw new RuntimeException("selectedKey is null");
            } else {
                selectedKeyList.add(selectedKey);

                // 중복 허용하지 않을 시, 가중치 풀에서 삭제
                if(!isDuplicated) {
                    this.removeEntry(selectedKey);
                }
            }
        }

        return selectedKeyList;
    }
}
```  

- 선출할 개수인 `needCount` 를 k, 전체 가중치의 개수를 n 이라고 할때, `ProtoWeightSelector` 방식의 시간복잡도는 % O(kn) % 이 된다.

### 테스트

```java
public class ProtoWeightSelectorTest {
    @Test
    public void selectKeyIsDuplicated_Once_선출된개수가오차범위안에들어온다() throws Exception {
        // given
        int[] weightArray = new int[]{
                10,
                20,
                30,
                40
        };
        int len = weightArray.length;
        double[] weightPercentageArray = Util.getPercentageArray(weightArray);
        System.out.println(Arrays.toString(weightPercentageArray));

        ProtoWeightSelector<Integer, WeightEntry<Integer>> selector = new ProtoWeightSelector<>();
        for (int i = 0; i < len; i++) {
            selector.addEntry(WeightEntry.<Integer>builder()
                    .key(i)
                    .weight(weightArray[i])
                    .build());
        }
        int selectCount = 100000;

        // when
        List<Integer> actual = selector.processSelectKey(selectCount, true);
        // 선출된 키 List 를 각 key 가 몇번 나왔는지 index 기반으로 몇번 나왔는지 카운트한다.
        int[] actualKeyCountArray = new int[len];
        for (Integer key : actual) {
            actualKeyCountArray[key]++;
        }

        // then
        int interval = (int)(selectCount * 0.01);
        assertThat(actualKeyCountArray[0], allOf(
                greaterThanOrEqualTo(10000 - interval),
                lessThanOrEqualTo(10000 + interval)
        ));
        assertThat(actualKeyCountArray[1], allOf(
                greaterThanOrEqualTo(20000 - interval),
                lessThanOrEqualTo(20000 + interval)
        ));
        assertThat(actualKeyCountArray[2], allOf(
                greaterThanOrEqualTo(30000 - interval),
                lessThanOrEqualTo(30000 + interval)
        ));
        assertThat(actualKeyCountArray[3], allOf(
                greaterThanOrEqualTo(40000 - interval),
                lessThanOrEqualTo(40000 + interval)
        ));
    }

    @Test
    public void selectKeyIsDuplicated_Multiple_선출된개수가오차범위안에들어온다() throws Exception {
        // given
        int loopCount = 10000;

        for (int k = 0; k < loopCount; k++) {
            int selectCount = 100000;
            int[] weightArray = new int[]{
                    10,
                    20,
                    30,
                    40
            };
            int len = weightArray.length;
            ProtoWeightSelector<Integer, WeightEntry<Integer>> selector = new ProtoWeightSelector<>();
            for (int i = 0; i < len; i++) {
                selector.addEntry(WeightEntry.<Integer>builder()
                        .key(i)
                        .weight(weightArray[i])
                        .build());
            }

            // when
            List<Integer> actual = selector.processSelectKey(selectCount, true);
            // 선출된 키 List 를 각 key 가 몇번 나왔는지 index 기반으로 몇번 나왔는지 카운트한다.
            int[] actualKeyCountArray = new int[len];
            for (Integer key : actual) {
                actualKeyCountArray[key]++;
            }

            // then
            int interval = (int)(selectCount * 0.01);
            assertThat(actual, hasSize(selectCount));
            assertThat(actualKeyCountArray[0], allOf(
                    greaterThanOrEqualTo(10000 - interval),
                    lessThanOrEqualTo(10000 + interval)
            ));
            assertThat(actualKeyCountArray[1], allOf(
                    greaterThanOrEqualTo(20000 - interval),
                    lessThanOrEqualTo(20000 + interval)
            ));
            assertThat(actualKeyCountArray[2], allOf(
                    greaterThanOrEqualTo(30000 - interval),
                    lessThanOrEqualTo(30000 + interval)
            ));
            assertThat(actualKeyCountArray[3], allOf(
                    greaterThanOrEqualTo(40000 - interval),
                    lessThanOrEqualTo(40000 + interval)
            ));
        }
    }

    @Test
    public void selectedKeyNotDuplicated_Once_가중치개수만큼_선출하면모두선출된다() throws Exception {
        // given
        int[] weightArray = new int[]{
                10,
                20,
                30,
                40
        };
        int len = weightArray.length;
        ProtoWeightSelector<Integer, WeightEntry<Integer>> selector = new ProtoWeightSelector<>();
        for (int i = 0; i < len; i++) {
            selector.addEntry(WeightEntry.<Integer>builder()
                    .key(i)
                    .weight(weightArray[i])
                    .build());
        }
        int selectCount = 4;

        // when
        List<Integer> actual = selector.processSelectKey(selectCount, false);

        // then
        assertThat(actual, hasSize(selectCount));
        assertThat(actual, hasItem(0));
        assertThat(actual, hasItem(1));
        assertThat(actual, hasItem(2));
        assertThat(actual, hasItem(3));
    }

    @Test
    public void selectKeyNotDuplicated_Multiple_1개씩여러번_선출된수가오차범위안에들어온다() throws Exception {
        // given
        int loopCount = 100000;
        int interval = (int) (loopCount * 0.01);
        int[] weightArray = new int[]{
                10,
                20,
                30,
                40
        };
        int len = weightArray.length;
        int[] actualKeyCountArray = new int[len];
        for (int k = 0; k < loopCount; k++) {
            ProtoWeightSelector<Integer, WeightEntry<Integer>> selector = new ProtoWeightSelector<>();
            for (int i = 0; i < len; i++) {
                selector.addEntry(WeightEntry.<Integer>builder()
                        .key(i)
                        .weight(weightArray[i])
                        .build());
            }
            int selectCount = 1;

            // when
            List<Integer> actual = selector.processSelectKey(selectCount, false);
            // 선출된 키 List 를 각 key 가 몇번 나왔는지 index 기반으로 몇번 나왔는지 카운트한다.
            for (Integer key : actual) {
                actualKeyCountArray[key]++;
            }
        }

        // then
        assertThat(actualKeyCountArray[0], allOf(
                greaterThanOrEqualTo(10000 - interval),
                lessThanOrEqualTo(10000 + interval)
        ));
        assertThat(actualKeyCountArray[1], allOf(
                greaterThanOrEqualTo(20000 - interval),
                lessThanOrEqualTo(20000 + interval)
        ));
        assertThat(actualKeyCountArray[2], allOf(
                greaterThanOrEqualTo(30000 - interval),
                lessThanOrEqualTo(30000 + interval)
        ));
        assertThat(actualKeyCountArray[3], allOf(
                greaterThanOrEqualTo(40000 - interval),
                lessThanOrEqualTo(40000 + interval)
        ));
    }

    @Test(timeout = 7000)
    public void selectKeyIsDuplicated_WeightCount_100000_TimeIn_7000() throws Exception{
        // given
        int weightCount = 100000;
        int[] weightArray = new int[weightCount];
        for(int i = 1; i < weightCount; i++) {
            weightArray[i - 1] = i * 10;
        }
        ProtoWeightSelector<Integer, WeightEntry<Integer>> selector = new ProtoWeightSelector<>();
        for(int i = 0; i < weightCount; i++) {
            selector.addEntry(WeightEntry.<Integer>builder()
                    .key(i)
                    .weight(weightArray[i])
                    .build());
        }
        int selectCount = weightCount / 10;

        // when
        List<Integer> actual = selector.processSelectKey(selectCount, true);

        // then
        assertThat(actual, hasSize(selectCount));
    }

    @Test(timeout = 7000)
    public void selectKeyNotDuplicated_WeightCount_100000_TimeIn_7000() throws Exception{
        // given
        int weightCount = 100000;
        int[] weightArray = new int[weightCount];
        for(int i = 1; i < weightCount; i++) {
            weightArray[i - 1] = i * 10;
        }
        ProtoWeightSelector<Integer, WeightEntry<Integer>> selector = new ProtoWeightSelector<>();
        for(int i = 0; i < weightCount; i++) {
            selector.addEntry(WeightEntry.<Integer>builder()
                    .key(i)
                    .weight(weightArray[i])
                    .build());
        }
        int selectCount = weightCount / 10;

        // when
        List<Integer> actual = selector.processSelectKey(selectCount, false);

        // then
        assertThat(actual, hasSize(selectCount));
    }
}
```  

## 초기 방식의 문제점
- `초기 방식` 의 문제점은 전체 풀에서 선출된 가중치가 빠지게 됐을 떄, 나머지 가중치들의 확률값이 변한다는 점에 있다.
- 아래와 같은 가중치와 가중치에 해당하는 확률 값이 있다.

	
	key|0|1|2|3
	---|---|---|---|---
	weight|10|20|30|40 
	probability|10%|20%|30%|40%
	
- `key` 1번의 가중치 값이 선출돼서 제외된다고 하면 아래와 같이 가중치 풀과 확률값이 변하게 된다.
	
	key|0|2|3
	---|---|---|---
	weight|10|30|40 
	probability|12.5%|37.5%|50%

- 변한 가중치 풀을 기반으로 다시 선출하는 것을 반복하게 된다.
- 여기서 알수 있는 점은 가중치의 풀을 변하게하지 않는다면, 선출 한번마다 가중치 풀을 순회해서 해당하는 가중치를 찾는 작업에 대한 성능을 개선시킬 수 있다.

## 개선된 방식
- `개선된 방식`은 `초기 방식의 문제점` 에서 언급했던, 가중치 풀을 변하게 하지 않고 문제를 해결하는 방법을 사용했다.




















































---
## Reference
