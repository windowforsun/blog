e--- 
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
- 여기서 가중치란 확률분포를 뜻한다.
- 아이템 뽑기, 아이템 강화, 상점 리스트 선출, 캐릭터 능력치 수치 등 다양한 기획에서 등장한다.
- 관련 개발을 하며 초기버전과 이를 개선한 방법에 대해 소개한다.
- 여기서 가중치라는 의미는 확률과도 밀접한 관련이 있다. 가중치는 전체 가중치를 가진 풀(Pool) 중에서 각 요소들이 얼마의 확률을 가지고 있는지를 의미한다.
- 가중치에 대한 확률이란, 각 요소의 가중치의 값이 바뀌거나 가중치의 요소가 추가/삭제가 될때마다 유동적으로 확률 값이 바뀌게 된다.
- 구현된 알고리즘의 테스트는 선출된 분포에 대한 정확성 테스트와 성능 테스트로 나눠 진행한다.
	- 선출 비율에 대한 정확성 테스트는 오차범위 $\pm0.2$ 로 진행한다.

### 예시 기획
- 정산마다 랭킹에 따른 가중치를 차등으로 부여한다.
- 랭킹에 대한 가중치를 기반으로 1등급 보상 q명, 2등급 보상 w명, 3등급 보상 e명 순으로 지급한다.
- 한번 보상에 선정된 유저는 제외된다.

### 예시 데이터
- 랜덤 선출에 사용할 확률분포(가중치)는 아래와 같고, 이를 `가중치 풀`이라고 한다.

	key|weight
	---|---
	0|10
	1|20
	2|30
	3|40
	
- 가중치 데이터의 누적 확률 분포를 구하면 아래와 같다.

	key|weight|accWeight
	---|---|---
	0|10|10
	1|20|30
	2|30|60
	3|40|100
	
- 4개의 가중치를 가지고 1개만 선출할때, 확률분포와 누적확률분포를 통해 확률을 구하면 아래와 같다.
	<!--- 확률분포(가중치) 를 경우의수 case 는 $ case = weight \over accWeight $-->
	- `weight`, 누적확률분포를 `accWeight` 라고 할때, 확률 p(percentage) 는 $ p = {weight \over accWeight} \times 100$ 으로 구할 수 있다.

	key|p
	---|---
	0|10%
	1|20%
	2|30%
	3|40%
	
- 이후 계산을 쉽게하기 위해서 기존 확률 p를 100 으로 나눠, 1에 대한 비율 r을 아래와 같이 표현한다.
	- 1에 대한 비율 r(ratio) 는 $ r = {weight \over accWeight} $으로 구할 수 있다.
	- 모든 비율에 대한 값은 소숫점 5자리에서 반올림 한다.

	key|r
	---|---
	0|0.1
	1|0.2
	2|0.3
	3|0.4


### 중복을 허용하지 않고, 여러개 선출할때의 확률분포
- 먼저 예시의 데이터를 바탕으로 예상되는 백분을 값을 구해본다.
- 도출된 결과값은 실제 소스코드의 테스트에서 실제 데이터로 사용된다.

#### 1개 선출하기
- 먼저 하나도 선출되지 않은 경우, 각 `key` 에 대한 비율 r은 아래와 같다.
	- n번째의 각 `key` 에대한 비율을 $ r_n $ 이라 표현하고, n 번째 각 `key` i 에대한 비율은 $ r_n(i) $ 라고 표현한다.
	

	key|$r_1$
	---|---
	0|0.1
	1|0.2
	2|0.3
	3|0.4
	
	- 1개만 선출할때 각 `key` 의 확률 p 는 $ p = r * 100 $ 이 된다.
- 1번째 선출 때 가지는, 각 `key` 의 평균 비율인 ar(average ratio) 는 아래와 같다.

	key|$ar_1$
	---|---
	0|0.1
	1|0.2
	2|0.3
	3|0.4

- 최종적으로 1개만 선출할 경우 각 `key` 가 선출될 확률은 아래와 같다.
	
	key|$p_1$
	---|---
	0|10%
	1|20%
	2|30%
	3|40%
	
#### 2개 선출하기
- 1번째 선출 되었을 떄, 2번째 선출에 대한 각 `key` 의 비율은 아래와 같다.

	선출 된 key|0의 비율|1의 비율|2의 비율|3의 비율
	---|---|---|---|---
	0|0|0.222|0.333|0.444
	1|0.125|0|0.375|0.5
	2|0.1429|0.2857|0|0.5714
	3|0.1667|0.3333|0.5|0
	
	- 1번째로 선출 된 `key` 가 i 일때, i 를 제외하고 다른 `key` 값들이 가지는 비율을 나태낸 표이다.
	
- 위 표에 `key` i 가 선출될 비율 까지 더하면 아래와 같다.

	선출 된 key|key 가 선출될 비율($kr_2$)|0의 비율|1의 비율|2의 비율|3의 비율
	---|---|---|---|---|---
	0|0.1|0|0.222|0.333|0.444
	1|0.2|0.125|0|0.375|0.5
	2|0.3|0.1429|0.2857|0|0.5714
	3|0.4|0.1667|0.3333|0.5|0
	
	- 1번째가 선출된 이후 고려할 경우의 수는 ${}_4 \mathrm{ C }_1$ 로 4가지가 될 수 있다.
	- `key 가 선출될 비율` 을 kr(key ratio) 이라고 한다.
	- `kr` 은 4가지 중 한가지의 경우를 사용해 2번째 선출확률을 사용하게 되는데, 그때 한가지 경우를 선택할 비율을 의미한다.
	- `kr` 은 1번째의 각 `key` 의 비율과 동일한 것을 알 수 있다.		
- 1번째에서 i번을 제외한 `key` 가 선출 되었다고 가정했을 때, i번이 2번째에 선출될 비율인 $r_2(i)$ 은 아래와 같이 구할 수 있다. (i = 0 이라고 가정)
	- $ r_2(0) = kr_2(0) \times 0 + kr_2(1) \times 0.125 + kr_2(2) \times 0.1429 + kr_2(3) \times 0.1667 $
	- $ r_2(0) = 0.1 \times 0 + 0.2 \times 0.125 + 0.3 \times 0.1429 + 0.4 \times 0.1667 = 0.13455 $
- 위의 수식으로 각 `key` 마다 2번째에 선출될 비율을 구하면 아래와 같다.

	key|$r_2$
	---|---
	0|0.1346
	1|0.2412‬
	2|0.3083
	3|0.3154
	
- 이제 1번째, 2번째 까지 모두 고려해 `key` i 가 선출될 평균 비율 $ar_2$ 는 아래와 같이 구할 수 있다.
	- $$ar_2(i) = \frac{r_1(i) + r_2(i)}{2}$$
	- $$ar_2(0) = \frac{0.1 + 0.13455}{2} = 0.1173 $$
	
	key|$ar_2$
	---|---
	0|0.1173
	1|0.2206
	2|0.3042
	3|0.3577‬

- 최종적으로 2개 선출할 경우 각 `key` 가 선출될 확률은 아래와 같다.
	
	key|$p_2$
	---|---
	0|11.73%
	1|22.06%
	2|30.42%
	3|35.77%

#### 3개 선출하기	
- 2번째 선출 까지 끝나고 3번째 선출할때, 나올 수 있는 경우의 수는 ${}_4 \mathrm{ C }_2$ 로 아래와 같이 6가지가 된다.

	선출 된 key|0의 비율|1의 비율|2의 비율|3의 비율
	---|---|---|---|---
	0,1|0|0|0.4286|0.5714
	0,2|0|0.3333|0|0.6667
	0,3|0|0.4|0.6|0
	1,2|0.2|0|0|0.8
	1,3|0.25|0|0.75|0
	2,3|0.3333|0.6667|0|0
	
- 3번째 선출에서는 6가지 중 하나의 비율로 하나를 선출하게 되는데, 그때 각 가지수가 가지는 비율은 아래와 같이 구할 수 있다.
	- 이미 선출된 `key` 는 각 2개씩으로 각 키를, z,x 라고 한다면 $ kr_3(z, x) = ar_1(z) \times {kr_2(z)일때 x 비율} + ar_1(x) \times {kr_2(x)일때 z 비율} $
- 위 수식을 바탕으로 선출된 `key` 의 비율을 추가하면 아래와 같다.

	선출 된 key|key 가 선출될 비율($kr_3$)|0의 비율|1의 비율|2의 비율|3의 비율
	---|---|---|---|---|---
	0,1|0.0472|0|0|0.4286|0.5714
	0,2|0.0762|0|0.3333|0|0.6667
	0,3|0.1111|0|0.4|0.6|0
	1,2|0.1607|0.2|0|0|0.8
	1,3|0.2333‬|0.25|0|0.75|0
	2,3|0.3714|0.3333|0.6667|0|0
	
- 1번째, 2번째에서 선출된 z,x 를 제외했을 때, i번이 3번째에 선출될 비율인 $r_3(i)$ 은 아래와 같이 구할 수 있다. (i = 0 이라고 가정, 0의 비율이 0인 경우 수식에서 제외한다.)
	- $r_3(0) = kr_3(1,2) \times 0.2 + kr_3(1,3) \times 0.25 + kr_3(2,3) \times 0.3333 $
	- $r_3(0) = 0.1607 \times 0.2 + 0.2333 \times 0.25 + 0.3714 \times 0.3333 = 0.2143$
- 위의 수식으로 각 `key` 마다 3번째에 선출될 비율을 구하면 아래와 같다.
	
	key|$r_3$
	---|---
	0|0.2143
	1|0.3174
	2|0.2619
	3|0.2063
	
- 이제 1번째, 2번째, 3번째 까지 모두 고려해 `key` i 가 선출될 평균 비율 $ar_3$ 는 아래와 같이 구할 수 있다.
	- $$ar_3(i) = \frac{r_1(i) + r_2(i) + r_3(i)}{3}$$
	- $$ar_3(0) = \frac{0.1 + 0.1346 + 0.2143}{3} = 0.1439$$
	
	key|$ar_3$
	---|---
	0|0.1496
	1|0.2529
	2|0.2901
	3|0.3072

- 최종적으로 3개 선출할 경우 각 `key` 가 선출될 확률은 아래와 같다.
	
	key|$p_3$
	---|---
	0|14.96%
	1|25.29%
	2|29.01%
	3|30.72%
	
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
- `weight` 는 가중치의 실제 값을 의미한다.

### WeightSelector

```java
public abstract class WeightSelector<K, V extends WeightEntry<K>> {
    protected long totalWeight;
    protected Random random;

    public WeightSelector() {
        this.totalWeight = 0;
        this.random = new Random();
    }

    public void addEntry(V entry) {
        this.totalWeight += entry.getWeight();
        this.add(entry);
    }

    public long getRandomWeight() {
        return Math.abs(this.random.nextLong()) % this.totalWeight;
    }

    public boolean checkProcess(int needCount, boolean isDuplicated) {
        boolean result = true;

        if(!isDuplicated && this.getEntrySize() < needCount) {
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

    protected abstract void add(V entry);

    public abstract int getEntrySize();

    protected abstract List<K> getSelectedKeyList(int needCount, boolean isDuplicated);
}
```  

- `WeightSelector` 는 가중치 정보(`WeightEntry`) 를 바탕으로 가중치를 기반으로 랜덤 선출하기 위해 필요한 메소드가 정의된 추상 클래스이다.
- 2개의 Generic 타입을 사용한다.
	- `K` : `WeightEntry` 에서 사용하는 가중치의 고유 키에 대한 타입
	- `V` : 가중치 정보를 나타내는 클래스의 타입으로 `WeightEntry` 의 하위 클래스여야 한다.
- 선출시에 사용하는 전체 가중치 정보를 저장하는 자료구조는 하위 클래스에서 정의한다.
- `totalWeight` 는 전체 가중치의 총 합을 나태내는 필드이다.
- `Random` 클래스를 통해 랜덤 값을 사용한다.
- `addEntry` 메소드는 `WeightEntry` 를 인자값으로 받아 `add()` 메소드를 사용해서 가중치 정보에 추가하는데, 추가할때 `totalWeight` 에 인자값의 가중치를 더해준다.
	- `add()` 메소드는 하위 클래스에서 구현한 내용대로, 하위 클래스에서 정의한 자료구조에 인자값의 가중치 정보를 저장한다.
- `getRandomWeight()` 메소드는 `totalWeight` 를 기반으로 선출할 랜덤 가중치를 뽑는다.
- `processSelectKey()` 메소드는 인자 값으로 선출할 개수와, 중복 허용 여부를 받아 `checkProcess()` 메소드와 `getSelectedKeyList()` 메소드를 사용해서 선출 된 키를 `List` 로 반환한다.
	- `getSelectedKeyList()` 메소드는 하위 클래스에서 구현한 내용대로 키를 선출해 `List` 형식으로 반환한다.
	- `checkProcess()` 메소드는 선출 전 유효성 검사를 수행한다.
	- `getSize()` 메소드는 하위 클래스에서 구현한 내용대로 현재 가중치의 개수를 리턴한다.

### Util

```java
public class Util {
    public static double[] getPercentageArray(List<Integer> keyList, int keyCount) {
        int[] keyCountArray = new int[keyCount];
        double[] percentageArray = new double[keyCount];

        for(Integer key : keyList) {
            keyCountArray[key]++;
        }

        double sum = Arrays.stream(keyCountArray).sum();
        for(int i = 0; i < keyCount; i++) {
            percentageArray[i] = (keyCountArray[i] / sum) * 100;
        }

        return percentageArray;
    }
}
```  

- `Util` 클래스의 `getPercentage()` 메소드는 확률을 반환해주는 정적 메서드이다.
- 인자 값으로 선출된 키의 리스트와 키의 종류 개수를 받으면 이를 비율의 백분율 값으로 환산해서 리턴한다.
- Test 시에 유틸로 사용된다.

#### Util 테스트

```java
public class UtilTest {
    @Test
    public void getPercentageArray_0_1개_100리턴() {
        // given
        List<Integer> list = new ArrayList<>(Arrays.asList(0));

        // when
        double[] actual = Util.getPercentageArray(list, 1);

        // then
        assertThat(actual[0], is(100d));
    }

    @Test
    public void getPercentageArray_0_1개_1_1개_50_50리턴() {
        // given
        List<Integer> list = new ArrayList<>(Arrays.asList(0, 1));

        // when
        double[] actual = Util.getPercentageArray(list, 2);

        // then
        assertThat(actual[0], is(50d));
        assertThat(actual[1], is(50d));
    }

    @Test
    public void getPercentageArray_0_1개_1_2개_33_66리턴() {
        // given
        List<Integer> list = new ArrayList<>(Arrays.asList(0, 1, 1));

        // when
        double[] actual = Util.getPercentageArray(list, 2);

        // then
        assertThat(actual[0], closeTo(33.3d, 0.1));
        assertThat(actual[1], closeTo(66.6d, 0.1));
    }

    @Test
    public void getPercentageArray_0_1개_1_1개_2_1개_33_33_33리턴() {
        // given
        List<Integer> list = new ArrayList<>(Arrays.asList(0, 1, 2));

        // when
        double[] actual = Util.getPercentageArray(list, 3);

        // then
        assertThat(actual[0], closeTo(33.3d, 0.1));
        assertThat(actual[1], closeTo(33.3d, 0.1));
        assertThat(actual[2], closeTo(33.3d, 0.1));
    }
}
```  

## 초기 방식
- 초기 개발버전에 사용했던 방식이고, 구현이 간단한 방법이다.
- 예시로 3개의 키를 선출해 본다.
- 아래와 같은 가중치 데이터가 있을 때, 모든 가중치의 합인 `totalWeight` 를 구한다.

	key|weight
	---|---
	0|10
	1|20
	2|30
	3|40
	
	- `totalWeight` 는 100 이 된다.
- 선출 할때 마다 0 ~ 99(`totalWeight`) 중 랜덤 값을 하나 뽑고, 뽑힌 값을 `randomWeight` 라고 한다.
- `randomWeight` 의 값이 50이라고 했을 때 가중치 풀을 차례대로 순회하며 `randomWeight` 보다 첫 번째로 큰 `weight` 의 `key` 를 구한다.
	- 이때 순회하며 `randomWeight` 보다 `weight` 값이 크지 않다면 `randomWeight` 에 `key` 에 해당하는 `weight` 값을 빼준다.

	key|weight|randomWeight
	---|---|---
	0|10|50
	1|20|40
	2|30|20
	3|40|x
	

- `key` 가 2인 `weight` 값 30 이 `randomWeight` 값 20보다 크기 때문에, 첫번째로 선출된 가중치의 `key` 는 2가 된다.
- 선출된 가중치를 지우면 아래와 같이 되고, `totalWeight` 는 70이 된다.

	key|weight
	---|---
	0|10
	1|20
	3|40
	
- 다시 0 ~ 69(`totalWeight`) 중 랜덤 값을 뽑아 `randomWeight` 의 값은 0 일때, 다시 가중치 풀을 순회하며 `randomWeight` 보다 크지만 가장 작은 값을 찾는다.

	key|weight|randomWeight
	---|---|---
	0|10|0
	1|20|x
	3|40|x
	
- `key` 0 의 `weight` 인 10 이 0(`randomWeight`) 보다 크기 때문에, 두번째 선출된 가중치의 `key` 는 0이 된다.
- 마지막으로 가중치를 지우면 아래와 같이 되고, `totalWeight` 는 60이 된다.

	key|weight
	---|---
	1|20
	3|40
	
- 마지막으로 0 ~ 59(`totalWeight`) 중 랜덤 값을 뽑아 `randomWeight` 의 값은 30 일때, 다시 가중치 풀을 순회하며 `randomWeight` 보다 크지만 가장 작은 값을 찾는다.

	key|weight|randomWeight
	---|---|---
	1|20|30
	3|40|10
	
- `key` 3 의 `weight` 인 40 이 10(`randomWeight`) 보다 크기 때문에, 마지막으로 선출된 가중치의 `key` 는 3이 되서 최종적으로 뽑힌 가중치의 키는 `2, 0, 3`(뽑힌 순서대로) 가 된다.

### 소스코드

![그림 1]({{site.baseurl}}/img/algorithm/practice_randomweightselector_1.png)

- 초기 방식의 `WeightSelector` 의 구현체는 전체 가중치의 개수를 n, 선출할 개수를 k 라고 할때 $O(kn),k<n$ 시간복잡도를 갖는다.

#### ProtoWeightSelector

```java
public class ProtoWeightSelector<K, V extends WeightEntry<K>> extends WeightSelector<K, V> {
    // Map 을 가중치 풀로 사용
    private Map<K, V> entryMap;

    public ProtoWeightSelector() {
        this.entryMap = new HashMap<>();
    }

    // key 에 해당하는 가중치를 삭제하고, totalWeight 에서도 차감
    protected void removeEntry(K key) {
        this.totalWeight -= this.entryMap.remove(key).getWeight();
    }

    @Override
    protected void add(V entry) {
        this.entryMap.put(entry.getKey(), entry);
    }

    @Override
    public int getEntrySize() {
        return entryMap.size();
    }

    @Override
    protected List<K> getSelectedKeyList(int needCount, boolean isDuplicated) {
        List<K> selectedKeyList = new LinkedList<>();
        // 가중치 풀
        Set<Map.Entry<K, V>> entrySet = this.entryMap.entrySet();
        long randomWeight, entryWeight;
        V selectedEntry;

        // 선출할 수만큼 반복
        for(int i = 0; i < needCount; i++) {
            // i 번째 선출시 사용할 랜덤 가중치 값
            randomWeight = this.getRandomWeight();
            selectedEntry = null;

            // 가중치 풀을 순회하며 랜덤 가중치 값 보다 큰 값 찾기
            for(Map.Entry<K, V> entry : entrySet) {
                entryWeight = entry.getValue().getWeight();

                if(randomWeight < entryWeight) {
                    // 랜덤 가중치 값보다 크면, 선택된 엔트리로
                    selectedEntry = entry.getValue();
                    break;
                } else {
                    // 랜덤 가중치 값보다 크지 않다면, i 번째 가중치 값만큼 차감
                    randomWeight -= entryWeight;
                }
            }

            if(selectedEntry == null) {
                throw new RuntimeException("selectedKey is null");
            } else {
                selectedKeyList.add(selectedEntry.getKey());

                // 중복 허용하지 않을 시, 가중치 풀에서 삭제
                if(!isDuplicated) {
                    this.removeEntry(selectedEntry.getKey());
                }
            }
        }

        return selectedKeyList;
    }
}
```  

### 테스트

```java
public class ProtoWeightSelectorTest {
    @Test
    public void 중복허용_1번수행_선출된키의비율이_오차범위에들어온다() throws Exception {
        // given
        int[] weightArray = new int[]{
                10,
                20,
                30,
                40
        };
        int len = weightArray.length;
        int selectCount = 1000000;
        WeightSelector<Integer, WeightEntry<Integer>> selector = new ProtoWeightSelector<>();
        for (int i = 0; i < len; i++) {
            selector.addEntry(WeightEntry.<Integer>builder()
                    .key(i)
                    .weight(weightArray[i])
                    .build());
        }

        // when
        List<Integer> actual = selector.processSelectKey(selectCount, true);
        double[] actualPercentageArray = Util.getPercentageArray(actual, len);

        // then
        assertThat(actual, hasSize(selectCount));
        assertThat(actualPercentageArray[0], closeTo(10, 0.2));
        assertThat(actualPercentageArray[1], closeTo(20, 0.2));
        assertThat(actualPercentageArray[2], closeTo(30, 0.2));
        assertThat(actualPercentageArray[3], closeTo(40, 0.2));
    }

    @Test
    public void 중복허용_여러번수행_매번선출된키의비율이_오차범위에들어온다() throws Exception {
        // given
        int loopCount = 100;
        int[] weightArray = new int[]{
                10,
                20,
                30,
                40
        };
        int len = weightArray.length;
        int selectCount = 1000000;

        for (int k = 0; k < loopCount; k++) {
            WeightSelector<Integer, WeightEntry<Integer>> selector = new ProtoWeightSelector<>();
            for (int i = 0; i < len; i++) {
                selector.addEntry(WeightEntry.<Integer>builder()
                        .key(i)
                        .weight(weightArray[i])
                        .build());
            }

            // when
            List<Integer> actual = selector.processSelectKey(selectCount, true);
            double[] actualPercentageArray = Util.getPercentageArray(actual, len);

            // then
            assertThat(actual, hasSize(selectCount));
            assertThat(actualPercentageArray[0], closeTo(10, 0.2));
            assertThat(actualPercentageArray[1], closeTo(20, 0.2));
            assertThat(actualPercentageArray[2], closeTo(30, 0.2));
            assertThat(actualPercentageArray[3], closeTo(40, 0.2));
        }
    }

    @Test
    public void 중복허용하지않음_한번수행_가중치개수만큼선출_모든가중치의키가선출된다() throws Exception {
        // given
        int[] weightArray = new int[]{
                10,
                20,
                30,
                40
        };
        int len = weightArray.length;
        WeightSelector<Integer, WeightEntry<Integer>> selector = new ProtoWeightSelector<>();
        for (int i = 0; i < len; i++) {
            selector.addEntry(WeightEntry.<Integer>builder()
                    .key(i)
                    .weight(weightArray[i])
                    .build());
        }
        int selectCount = len;

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
    public void 중복허용하지않음_여러번수행_수행마다1개선출_종합적으로선출된키의비율이_오차범위에들어온다() throws Exception {
        // given
        int loopCount = 1000000;
        int[] weightArray = new int[]{
                10,
                20,
                30,
                40
        };
        int len = weightArray.length;
        List<Integer> actual = new LinkedList<>();
        int selectCount = 1;

        // when
        for (int k = 0; k < loopCount; k++) {
            WeightSelector<Integer, WeightEntry<Integer>> selector = new ProtoWeightSelector<>();
            for (int i = 0; i < len; i++) {
                selector.addEntry(WeightEntry.<Integer>builder()
                        .key(i)
                        .weight(weightArray[i])
                        .build());
            }

            actual.addAll(selector.processSelectKey(selectCount, false));
        }

        // then
        double[] actualPercentageArray = Util.getPercentageArray(actual, len);
        assertThat(actualPercentageArray[0], closeTo(10, 0.2));
        assertThat(actualPercentageArray[1], closeTo(20, 0.2));
        assertThat(actualPercentageArray[2], closeTo(30, 0.2));
        assertThat(actualPercentageArray[3], closeTo(40, 0.2));
    }

    @Test
    public void 중복허용하지않음_여러번수행_수행마다2개선출_종합적으로선출된키의비율이_오차범위에들어온다() throws Exception {
        // given
        int loopCount = 1000000;
        int selectCount = 2;
        int[] weightArray = new int[]{
                10,
                20,
                30,
                40
        };
        int len = weightArray.length;
        List<Integer> actual = new LinkedList<>();

        // when
        for (int k = 0; k < loopCount; k++) {
            WeightSelector<Integer, WeightEntry<Integer>> selector = new ProtoWeightSelector<>();
            for (int i = 0; i < len; i++) {
                selector.addEntry(WeightEntry.<Integer>builder()
                        .key(i)
                        .weight(weightArray[i])
                        .build());
            }

            actual.addAll(selector.processSelectKey(selectCount, false));
        }

        // then
        double[] actualPercentageArray = Util.getPercentageArray(actual, len);
        assertThat(actualPercentageArray[0], closeTo(11.73, 0.2));
        assertThat(actualPercentageArray[1], closeTo(22.06, 0.2));
        assertThat(actualPercentageArray[2], closeTo(30.42, 0.2));
        assertThat(actualPercentageArray[3], closeTo(35.77, 0.2));
    }

    @Test
    public void 중복허용하지않음_여러번수행_수행마다3개선출_종합적으로선출된키의비율이_오차범위에들어온다() throws Exception {
        // given
        int loopCount = 1000000;
        int selectCount = 3;
        int[] weightArray = new int[]{
                10,
                20,
                30,
                40
        };
        int len = weightArray.length;
        List<Integer> actual = new LinkedList<>();

        // when
        for (int k = 0; k < loopCount; k++) {
            WeightSelector<Integer, WeightEntry<Integer>> selector = new ProtoWeightSelector<>();
            for (int i = 0; i < len; i++) {
                selector.addEntry(WeightEntry.<Integer>builder()
                        .key(i)
                        .weight(weightArray[i])
                        .build());
            }

            actual.addAll(selector.processSelectKey(selectCount, false));
        }

        // then
        double[] actualPercentageArray = Util.getPercentageArray(actual, len);
        assertThat(actualPercentageArray[0], closeTo(14.96, 0.2));
        assertThat(actualPercentageArray[1], closeTo(25.29, 0.2));
        assertThat(actualPercentageArray[2], closeTo(29.01, 0.2));
        assertThat(actualPercentageArray[3], closeTo(30.72, 0.2));
    }

    @Test(timeout = 25000)
    public void 중복허용_100000개가중치중_절반선출_25000밀리초안에수행된다() throws Exception {
        // given
        int weightCount = 100000;
        int[] weightArray = new int[weightCount];
        for (int i = 1; i < weightCount; i++) {
            weightArray[i - 1] = i * 10;
        }
        WeightSelector<Integer, WeightEntry<Integer>> selector = new ProtoWeightSelector<>();
        for (int i = 0; i < weightCount; i++) {
            selector.addEntry(WeightEntry.<Integer>builder()
                    .key(i)
                    .weight(weightArray[i])
                    .build());
        }
        int selectCount = weightCount / 2;

        // when
        List<Integer> actual = selector.processSelectKey(selectCount, true);

        // then
        assertThat(actual, hasSize(selectCount));
    }

    @Test(timeout = 25000)
    public void 중복허용하지않음_100000개가중치중_절반선출_25000밀리초안에수행된다() throws Exception {
        // given
        int weightCount = 100000;
        int[] weightArray = new int[weightCount];
        for (int i = 1; i < weightCount; i++) {
            weightArray[i - 1] = i * 10;
        }
        WeightSelector<Integer, WeightEntry<Integer>> selector = new ProtoWeightSelector<>();
        for (int i = 0; i < weightCount; i++) {
            selector.addEntry(WeightEntry.<Integer>builder()
                    .key(i)
                    .weight(weightArray[i])
                    .build());
        }
        int selectCount = weightCount / 2;

        // when
        List<Integer> actual = selector.processSelectKey(selectCount, false);

        // then
        assertThat(actual, hasSize(selectCount));
    }
}
```  

- `ProtoWeightSelector` 의 테스트 결과 중복을 허용하는 경우와 중복을 허용하지 않는 경우 1개, 2개, 3개의 선출 일떄 모두 도출됐던 확률 값(오차범위 $\pm0.2$)을 만족한다.
- 성능 테스트로 100000개의 가중치 풀에서 절반인 50000개를 중복을 허용하고 선출할 때, 25초(25000밀리초) 가 걸리는 것을 확인 할 수 있다.
- 성능 테스트로 100000개의 가중치 풀에서 절반인 50000개를 중복을 허용하지 않고 선출할 때, 25초(25000밀리초) 가 걸리는 것을 확인 할 수 있다.

## 초기 방식의 문제점
- `초기 방식` 의 문제점은 전체 풀에서 선출된 가중치가 빠지게 됐을 떄, 나머지 가중치들의 전체 비율이 변하게 된다는 점이다.
- 아래와 같은 가중치와 가중치와 비율값이 있다.

	key|weight|r
	---|---|---
	0|10|0.1
	1|20|0.2
	2|30|0.3
	3|40|0.4
	
	
- `key` 1번의 가중치 값이 선출돼서 제외된다고 하면 아래와 같이 가중치 풀과 비율값이 변하게 된다.
	
	key|weight|r
	---|---|---
	0|10|0.125
	2|30|0.375
	3|40|0.5714

- 다음 선출시 위 표에 나와있는 비율을 기반으로 선출을 해야 되기 때문에 한번 전체 풀을 다시 조정해주는 연산이 들어가야한다.
	- 선출 수(k), 가중치 수(n)
	- 가중치 풀이 변할때마다(선출 1번당) 전체 풀에 대한 조정을 해야 하는데, $k<n$ 이기 때문에 가중치 풀을 조정해주는 연산
- 여기서 알 수 있는 점은 선출에 따라 가중치 풀이 변하게 될떄(비율이 변할때), 이를 조정해주는 연산을 개선할 경우 성능적으로 개선이 가능하다.

## 개선된 방식
- `개선된 방식`은 `초기 방식의 문제점` 에서 언급했던, 선출마다 가중치 풀 변화에 대한 조정을 개선하는 방식으로 진행했다.

### 방식 설명
- 방식 설명을 할때는 잠깐 다른 데이터로 진행한다.
- 아래와 같은 가중치 데이터가 있고, 가중치의 합은 55 다.

	key|weight|min|max|selected
	---|---|---|---|---
	0|1|0|0|false
	1|2|1|2|false
	2|3|3|5|false
	3|4|6|9|false
	4|5|10|14|false
	5|6|15|20|false
	6|7|21|27|false
	7|8|28|35|false
	8|9|36|44|false
	9|10|45|54|false
	
	- `selected` 는 해당 가중치가 선출됐는지에 대한 플래그 값이다.
- 1번째로 0~54 랜덤 값 중 4가 나왔다. `key` 2를 선출하면 아래와 같다.

	key|weight|min|max|selected
	---|---|---|---|---
	0|1|0|0|false
	1|2|1|2|false
	2|3|3|5|true
	3|4|6|9|false
	4|5|10|14|false
	5|6|15|20|false
	6|7|21|27|false
	7|8|28|35|false
	8|9|36|44|false
	9|10|45|54|false

	- `key` 2의 `selected` 에 `true` 값을 설정되었다. 이는 가중치 풀에서 삭제되었다는 것을 의미한다.
	- 가중치 합이 $55-3=52$ 로 변경된다.
- 2번째로 0~51 랜덤값 중 21 이 나왔다. 1번째 선출에서는 전과 같은 방식으로 선출했지만 2번째 부터는 아래와 같은 조건이 있다.
	- 가중치의 랜덤값이 선출된 이후, i번째 가중치의 범위는 i보다 작은 가중치 중 선택된 값들을 더해 가중치의 범위에 적용해준다.
	- `selected` 의 값이 `true` 라면 가중치 풀에서 제외 된다.
	- 위 표를 예시로 들면, `key` 2보다 큰 3~9 `key` 들의 `min`, `max` 의 값에 2의 가중치 값을 각각 차감해 주고 범위를 사용한다.
	- `min`, `max` 의 값을 차감시켜 사용한다는 것은 값을 매번 갱신시키는 것이 아니라, i 보다 작은 선출된 가중치의 합을 매번 구해 사용한다.
- 위 가중치 풀에서 14에 해당되는 `key` 는 5가 된다. $min = 15-3 = 13$, $max = 20-3 = 18$ 로 범위가 수정되기 때문이다.
	- 이전에 선택된 `key` 2 만큼 차감하고 범위를 비교한다.

	key|weight|min|max|selected
	---|---|---|---|---
	0|1|0|0|false
	1|2|1|2|false
	2|3|3|5|true
	3|4|6|9|false
	4|5|10|14|false
	5|6|15|20|true
	6|7|21|27|false
	7|8|28|35|false
	8|9|36|44|false
	9|10|45|54|false

	- 가중치 합이 $52-6=46$ 로 변경된다.
- 3번째로 0~45 랜덤값 중 27 이 나오면, `key` 8이 해당된다. $min = 36-(3+6) = 27$, $max = 44-(3+6) =35$ 이기 때문이다.
	- 이전에 선택된 `key` 2, 5 만큼 차감하고 범위를 비교한다.
	
	key|weight|min|max|selected
	---|---|---|---|---
	0|1|0|0|false
	1|2|1|2|false
	2|3|3|5|true
	3|4|6|9|false
	4|5|10|14|false
	5|6|15|20|true
	6|7|21|27|false
	7|8|28|35|false
	8|9|36|44|true
	9|10|45|54|false

	- 가중치 합이 $46-9=37$ 로 변경된다.
- 4번째로 0~36 랜덤값 중 4 가 나오면, `key` 4가 해당된다. $min = 5-3 = 2$, $max = 10-2 =8$ 이기 때문이다.
	- 이전에 선택된 `key` 2 만큼 차감하고 범위를 비교한다.

	key|weight|min|max|selected
	---|---|---|---|---
	0|1|0|0|false
	1|2|1|2|false
	2|3|3|5|true
	3|4|6|9|false
	4|5|10|14|false
	5|6|15|20|true
	6|7|21|27|false
	7|8|28|35|false
	8|9|36|44|true
	9|10|45|54|false
	
- 이후 소개할 실제 코드에서는 자신보다 `key` 값으로 작은 선택된 모든 가중치의 합을 구하기 위해 `SegmentTree` 를 사용했다.
- `SegmentTree` 는 구간에 합을 $O(\log_2 N)$ 만에 구할 수 있어서, 기존 $O(N)$ 보다 성능적으로 이점이 크다.

### 소스코드

![그림 1]({{site.baseurl}}/img/algorithm/practice_randomweightselector_2.png)

- 개선된 방식의 `WeightSelector` 구현채는 전체 가중치의 수를 n, 선출할 수를 k 로한다면 $O(k\log_2 n),k<n$ 의 시간 복잡도를 갖는다.

#### SegmentTree

```java
public class SegmentTree {
    private long[] dataArray;
    private int dataSize;
    private long[] segmentTreeArray;
    private int treeSize;
    private int dataStartIndex;

    public SegmentTree(long[] dataArray) {
        this.dataArray = dataArray;
        this.dataSize = this.dataArray.length;
        this.dataStartIndex = this.getTreeSize(this.dataSize);
        this.treeSize = this.dataStartIndex * 2;
        this.segmentTreeArray = new long[this.treeSize];

        for(int i = this.dataStartIndex; i < this.dataStartIndex + this.dataSize; i++) {
            this.segmentTreeArray[i] = this.dataArray[i - this.dataStartIndex];
        }

        this.initSegmentTree();
    }

    public void initSegmentTree() {
        for(int i = this.treeSize - 1; i > 0; i -= 2) {
            this.segmentTreeArray[i / 2] = this.segmentTreeArray[i] + this.segmentTreeArray[i - 1];
        }
    }

    public void update(int index, long value) {
        index += this.dataStartIndex;
        this.segmentTreeArray[index] = value;

        while(index > 1) {
            index /= 2;
            this.segmentTreeArray[index] = this.segmentTreeArray[index * 2] + this.segmentTreeArray[index * 2 + 1];
        }
    }

    public long getSum(int start, int end) {
        long sum = 0;
        if(end < 0) {
            return sum;
        }
        start += this.dataStartIndex;
        end += this.dataStartIndex;

        while(start < end) {
            if(start % 2 == 1) {
                sum += this.segmentTreeArray[start];
                start++;
            }

            if(end % 2 == 0) {
                sum += this.segmentTreeArray[end];
                end--;
            }

            start /= 2;
            end /= 2;
        }

        if(start == end) {
            sum += this.segmentTreeArray[start];
        }

        return sum;
    }

    public int getTreeSize(int dataSize) {
        int result = 1;

        while(result < dataSize) {
            result <<= 1;
        }

        return result;
    }
}
```  

- `SegmentTree` 클래스에 대한 자세한 설명은 [여기]({{site.baseurl}}{% link _posts/algorithm/2019-06-12-algorithm-concept-segmenttree.md %}) 에서 확인 가능하다.


#### AdvancedWeightEntry

```java
@Getter
@Setter
public class AdvancedWeightEntry<K> extends WeightEntry<K> {
    // 가중치 범위 중 최소 가중치
    protected long minWeight;
    // 가중치 버위 중 최대 가중치
    protected long maxWeight;
    // 가중치 풀의 초기 인덱스
    protected int index;


    @Builder(builderMethodName = "advancedWeightEntryBuilder")
    public AdvancedWeightEntry(K key, long weight, long minWeight, long maxWeight, int index) {
        super(key, weight);
        this.minWeight = minWeight;
        this.maxWeight = maxWeight;
        this.index = index;
    }
}
```  

- `minWeight` 는 해당 가중치의 범위중 최소 가중치, `maxWeight` 는 가중치 범위중 최대 가중치를 의미한다.
- `index` 는 `AdvancedWeightSelector` 클래스에서 사용하는 가중치 풀의 자료구조 인덱스를 의미하고, 이후 `SegmentTree` 에서도 사용된다.

#### AdvancedWeightSelector

```java
public class AdvancedWeightSelector<K, V extends AdvancedWeightEntry<K>> extends WeightSelector<K, V> {
    // List 를 가중치 풀로 사용
    private List<V> entryList;
    // 구간합을 구하기 위한 세그먼트 트리
    private SegmentTree segmentTree;

    public AdvancedWeightSelector(int entrySize) {
        super();
        this.entryList = new ArrayList<>(entrySize);
        this.segmentTree = new SegmentTree(new long[entrySize]);
    }

    @Override
    protected void add(V entry) {
        this.entryList.add(entry);
        // 현재 가중치 풀에서의 인덱스를 entry 에 설정
        entry.setIndex(this.entryList.size() - 1);
    }

    @Override
    public int getEntrySize() {
        return this.entryList.size();
    }

    @Override
    public void addEntry(V entry) {
        // entry 에 minWeight, maxWeight 를 설정
        entry.setMinWeight(this.totalWeight);
        super.addEntry(entry);
        entry.setMaxWeight(this.totalWeight - 1);
    }

    @Override
    protected List<K> getSelectedKeyList(int needCount, boolean isDuplicated) {
        List<K> selectedKeyList = new LinkedList<>();
        long randomWeight;
        int randomWeightIndex;
        V randomEntry;

        // 선출 개수만큼 반복
        for (int i = 0; i < needCount; i++) {
            // i 번째 선출시 사용할 랜덤 가중치 값
            randomWeight = this.getRandomWeight();
            // 가중치 풀에서 randomWeight에 해당하는 가중치의 인덱스를 검색
            randomWeightIndex = this.findEntryIndexByWeight(randomWeight, isDuplicated);
            randomEntry = this.entryList.get(randomWeightIndex);

            // 중복을 허용하지 않는 경우
            if(!isDuplicated) {
                // 선출된 entry 의 가중치를 totalWeight 에서 차감
                this.totalWeight -= randomEntry.getWeight();
                // 선출된 entry 의 가중치 정보를 index, weight 를 바탕으로 segment tree 갱신
                this.segmentTree.update(randomEntry.getIndex(), randomEntry.getWeight());
                // 선출된 entry 가중치 풀에서 삭제
                this.entryList.remove(randomWeightIndex);
            }

            // 선출된 key 추가
            selectedKeyList.add(randomEntry.getKey());
        }

        return selectedKeyList;
    }

    public int findEntryIndexByWeight(long weight, boolean isDuplicated) {
        Comparator comparator;

        // 중복 허용할때와 하지 않을 때 다른 정렬 기준을 사용
        if(isDuplicated) {
            comparator = new AdvancedDuplicatedComparator();
        } else {
            comparator = new AdvancedNotDuplicatedComparator(this.segmentTree);
        }

        // 이진 탐색으로 가중치 풀에서 검색해서 인덱스 리턴
        return Collections.binarySearch(this.entryList,
                // 검색 하는 키값인 randomWeight 값을 min, max 값으로 넣어 검색
                V.advancedWeightEntryBuilder()
                        .minWeight(weight)
                        .maxWeight(weight)
                        .build(),
                comparator);
    }
}
```  

#### AdvancedDuplicatedComparator 

```java
public class AdvancedDuplicatedComparator implements Comparator<AdvancedWeightEntry> {
    @Override
    public int compare(AdvancedWeightEntry o1, AdvancedWeightEntry o2) {
        int result = 0;
        long o1Min = o1.getMinWeight(), o1Max = o1.getMaxWeight(), o2Min = o2.getMinWeight(), o2Max = o2.getMaxWeight();

        // 최소, 최대 가중치가 모두 크면 크고, 모두 작으면 작다

        if (o1Min > o2Min && o1Max > o2Max) {
            result = 1;
        } else if (o1Min < o2Min && o1Max < o2Max) {
            result = -1;
        }

        return result;
    }
}
```  

#### AdvancedNotDuplicatedComparator

```java
public class AdvancedNotDuplicatedComparator implements Comparator<AdvancedWeightEntry> {
    private SegmentTree segmentTree;

    public AdvancedNotDuplicatedComparator(SegmentTree segmentTree) {
        this.segmentTree = segmentTree;
    }

    @Override
    public int compare(AdvancedWeightEntry o1, AdvancedWeightEntry o2) {
        int result = 0;
        long o1Min = o1.getMinWeight(), o1Max = o1.getMaxWeight(), o2Min = o2.getMinWeight(), o2Max = o2.getMaxWeight();

        // entry 의 key 가 null 인것은 찾으려는 randomWeight 를 뜻함
        if(o1.getKey() != null) {
            long offset1 = segmentTree.getSum(0, o1.getIndex() - 1);
            // randomWeight 가 아닌 entry 에는 자신보다 작은 키값중 선출된 가중치를 더한 값을 범위 가중치에서 차감
            o1Min -= offset1;
            o1Max -= offset1;
        }

        if(o2.getKey() != null) {
            long offset2 = segmentTree.getSum(0, o2.getIndex() - 1);
            o2Min -= offset2;
            o2Max -= offset2;
        }

        if (o1Min > o2Min && o1Max > o2Max) {
            result = 1;
        } else if (o1Min < o2Min && o1Max < o2Max) {
            result = -1;
        }

        return result;
    }
}
```  

### 테스트

```java
public class AdvancedWeightSelectorTest {
	@Test
    public void 중복허용_1번수행_선출된키의비율이_오차범위에들어온다() throws Exception{
        // given
        int[] weightArray = new int[]{
                10,
                20,
                30,
                40
        };
        int len = weightArray.length;
        int selectCount = 1000000;
        WeightSelector<Integer, AdvancedWeightEntry<Integer>> selector = new AdvancedWeightSelector<>(len);
        for (int i = 0; i < len; i++) {
            selector.addEntry(AdvancedWeightEntry.<Integer>advancedWeightEntryBuilder()
                    .key(i)
                    .weight(weightArray[i])
                    .build());
        }

        // when
        List<Integer> actual = selector.processSelectKey(selectCount, true);
        double[] actualPercentageArray = Util.getPercentageArray(actual, len);

        // then
        assertThat(actual, hasSize(selectCount));
        assertThat(actualPercentageArray[0], closeTo(10, 0.2));
        assertThat(actualPercentageArray[1], closeTo(20, 0.2));
        assertThat(actualPercentageArray[2], closeTo(30, 0.2));
        assertThat(actualPercentageArray[3], closeTo(40, 0.2));
    }

    @Test
    public void 중복허용_여러번수행_매번선출된키의비율이_오차범위에들어온다() throws Exception {
        // given
        int loopCount = 100;
        int[] weightArray = new int[]{
                10,
                20,
                30,
                40
        };
        int len = weightArray.length;
        int selectCount = 1000000;

        for(int k = 0; k < loopCount; k++) {
            WeightSelector<Integer, AdvancedWeightEntry<Integer>> selector = new AdvancedWeightSelector<>(len);
            for (int i = 0; i < len; i++) {
                selector.addEntry(AdvancedWeightEntry.<Integer>advancedWeightEntryBuilder()
                        .key(i)
                        .weight(weightArray[i])
                        .build());
            }

            // when
            List<Integer> selectedKeyList = selector.processSelectKey(selectCount, true);
            int[] selectedKeyCountArray = new int[len];
            for (Integer key : selectedKeyList) {
                selectedKeyCountArray[key]++;
            }

            // when
            List<Integer> actual = selector.processSelectKey(selectCount, true);
            double[] actualPercentageArray = Util.getPercentageArray(actual, len);

            // then
            assertThat(actual, hasSize(selectCount));
            assertThat(actualPercentageArray[0], closeTo(10, 0.2));
            assertThat(actualPercentageArray[1], closeTo(20, 0.2));
            assertThat(actualPercentageArray[2], closeTo(30, 0.2));
            assertThat(actualPercentageArray[3], closeTo(40, 0.2));
        }
    }

    @Test
    public void 중복허용하지않음_한번수행_가중치개수만큼선출_모든가중치의키가선출된다() throws Exception {
        // given
        int[] weightArray = new int[]{
                10,
                20,
                30,
                40
        };
        int len = weightArray.length;
        WeightSelector<Integer, AdvancedWeightEntry<Integer>> selector = new AdvancedWeightSelector<>(len);
        for (int i = 0; i < len; i++) {
            selector.addEntry(AdvancedWeightEntry.<Integer>advancedWeightEntryBuilder()
                    .key(i)
                    .weight(weightArray[i])
                    .build());
        }
        int selectCount = len;

        // when
        List<Integer> selectedKeyList = selector.processSelectKey(selectCount, false);
        int[] selectedKeyCountArray = new int[len];
        for (Integer key : selectedKeyList) {
            selectedKeyCountArray[key]++;
        }

        // then
        assertThat(selectedKeyList, hasSize(selectCount));
    }

    @Test
    public void 중복허용하지않음_여러번수행_수행마다1개선출_종합적으로선출된키의비율이_오차범위에들어온다() throws Exception{
        // given
        int loopCount = 1000000;
        int[] weightArray = new int[]{
                10,
                20,
                30,
                40
        };
        int len = weightArray.length;
        List<Integer> actual = new LinkedList<>();
        int selectCount = 1;

        // when
        for(int k = 0; k < loopCount; k++) {
            WeightSelector<Integer, AdvancedWeightEntry<Integer>> selector = new AdvancedWeightSelector<>(len);
            for (int i = 0; i < len; i++) {
                selector.addEntry(AdvancedWeightEntry.<Integer>advancedWeightEntryBuilder()
                        .key(i)
                        .weight(weightArray[i])
                        .build());
            }

            actual.addAll(selector.processSelectKey(selectCount, false));
        }

        // then
        double[] actualPercentageArray = Util.getPercentageArray(actual, len);
        assertThat(actualPercentageArray[0], closeTo(10, 0.2));
        assertThat(actualPercentageArray[1], closeTo(20, 0.2));
        assertThat(actualPercentageArray[2], closeTo(30, 0.2));
        assertThat(actualPercentageArray[3], closeTo(40, 0.2));
    }

    @Test
    public void 중복허용하지않음_여러번수행_수행마다2개선출_종합적으로선출된키의비율이_오차범위에들어온다() throws Exception {
        // given
        int loopCount = 1000000;
        int selectCount = 2;
        int[] weightArray = new int[]{
                10,
                20,
                30,
                40,
        };
        int len = weightArray.length;
        List<Integer> actual = new LinkedList<>();

        // when
        for(int k = 0; k < loopCount; k++) {
            WeightSelector<Integer, AdvancedWeightEntry<Integer>> selector = new AdvancedWeightSelector<>(len);
            for (int i = 0; i < len; i++) {
                selector.addEntry(AdvancedWeightEntry.<Integer>advancedWeightEntryBuilder()
                        .key(i)
                        .weight(weightArray[i])
                        .build());
            }

            actual.addAll(selector.processSelectKey(selectCount, false));
        }

        // then
        double[] actualPercentageArray = Util.getPercentageArray(actual, len);
        assertThat(actualPercentageArray[0], closeTo(11.73, 0.2));
        assertThat(actualPercentageArray[1], closeTo(22.06, 0.2));
        assertThat(actualPercentageArray[2], closeTo(30.42, 0.2));
        assertThat(actualPercentageArray[3], closeTo(35.77, 0.2));
    }

    @Test
    public void 중복허용하지않음_여러번수행_수행마다3개선출_종합적으로선출된키의비율이_오차범위에들어온다() throws Exception {
        // given
        int loopCount = 1000000;
        int selectCount = 3;
        int[] weightArray = new int[]{
                10,
                20,
                30,
                40
        };
        int len = weightArray.length;
        List<Integer> actual = new LinkedList<>();

        // when
        for(int k = 0; k < loopCount; k++) {
            WeightSelector<Integer, AdvancedWeightEntry<Integer>> selector = new AdvancedWeightSelector<>(len);
            for (int i = 0; i < len; i++) {
                selector.addEntry(AdvancedWeightEntry.<Integer>advancedWeightEntryBuilder()
                        .key(i)
                        .weight(weightArray[i])
                        .build());
            }

            actual.addAll(selector.processSelectKey(selectCount, false));
        }

        // then
        double[] actualPercentageArray = Util.getPercentageArray(actual, len);
        assertThat(actualPercentageArray[0], closeTo(14.96, 0.2));
        assertThat(actualPercentageArray[1], closeTo(25.29, 0.2));
        assertThat(actualPercentageArray[2], closeTo(29.01, 0.2));
        assertThat(actualPercentageArray[3], closeTo(30.72, 0.2));
    }

    @Test(timeout = 100)
    public void 중복허용_100000개가중치중_절반선출_100밀리초안에수행된다() throws Exception {
        // given
        int weightCount = 100000;
        int[] weightArray = new int[weightCount];
        for(int i = 1; i <= weightCount; i++) {
            weightArray[i - 1] = i * 10;
        }
        WeightSelector<Integer, AdvancedWeightEntry<Integer>> selector = new AdvancedWeightSelector<>(weightCount);
        for (int i = 0; i < weightCount; i++) {
            selector.addEntry(AdvancedWeightEntry.<Integer>advancedWeightEntryBuilder()
                    .key(i)
                    .weight(weightArray[i])
                    .build());
        }
        int selectCount = weightCount / 2;

        // when
        List<Integer> actual = selector.processSelectKey(selectCount, true);

        // then
        assertThat(actual, hasSize(selectCount));
    }

    @Test(timeout = 350)
    public void 중복허용하지않음_100000개가중치중_절반선출_350밀리초안에수행된다() throws Exception {
        // given
        int weightCount = 100000;
        int[] weightArray = new int[weightCount];
        for(int i = 0; i < weightCount; i++) {
            weightArray[i] = i + 1;
        }
        WeightSelector<Integer, AdvancedWeightEntry<Integer>> selector = new AdvancedWeightSelector<>(weightCount);
        for (int i = 0; i < weightCount; i++) {
            selector.addEntry(AdvancedWeightEntry.<Integer>advancedWeightEntryBuilder()
                    .key(i)
                    .weight(weightArray[i])
                    .build());
        }
        int selectCount = weightCount / 2;

        // when
        List<Integer> actual = selector.processSelectKey(selectCount, false);

        // then
        assertThat(actual, hasSize(selectCount));
    }
}
```  

- `AdvancedWeightSelector` 의 테스트 결과 중복을 허용하는 경우와 중복을 허용하지 않는 경우 1개, 2개, 3개의 선출 일떄 모두 도출됐던 확률 값(오차범위 $\pm0.2$)을 만족한다.
- 성능 테스트로 100000개의 가중치 풀에서 절반인 50000개를 중복을 허용하고 선출할 때, 0.1초(100밀리초) 가 소요되는 결과로 `ProtoWeightSelector` 와 비교했을 때 250배 정도 성능이 개선되었다.
- 성능 테스트로 100000개의 가중치 풀에서 절반인 50000개를 중복을 허용하지 않고 선출할 때, 0.35초(350밀리초) 가 소요되는 결과로 `ProtoWeightSelector` 와 비교했을 때 75배 정도 성능이 개선되었다.

---
## Reference
