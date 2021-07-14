# Redis 핵심정리
## 1. Redis 정의

Redis 는 고성능 key-value 데이터 저장소인 NoSQL(Not Only SQL) 이다.</br>
String, Hash, List, Set, Sorted Set, Bitmap, HyperLogLog 와 같은 다양한 데이터 타입을 제공하기 때문에 데이터 구조 서버라고도 부른다.</br>
기본적으로 모든 데이터를 메모리에 저장하므로 읽기와 쓰기 모두 빠르다.</br>
설정을 통해 디스크에도 저장이 가능한데 아래 두가지 방법이 있다.</br>
1. snapshotting : 저장된 데이터를 바이너리 형태로 스냅샷을 생성
2. journaling : 시간에 걸쳐 실행된 모든 커멘드를 순서대로 저장하여 사람이 읽을 수 있는 파일로 생성

또한 key expiration(키 만료) 와 transaction(트랜잭션), public/subscribe(게시/구독) 기능을 설정할 수 있다.

### redis-server
레디스의 실제 데이터 저장소이다.</br>
클러스터 모드 또는 독립 실행형(standalone) 모드로 실행될 수 있다.</br>

### redis-cli
모든 레디스 커멘드를 실행할 수 있는 커멘드 라인 인터페이스 이다.

## 2. 레디스 데이터 타입

데이터 타입을 이해하면 성능과 효율 측면에서 많은 이득을 볼 수 있다.

### String (문자열)

텍스트(XML, JSON, HTML, 원문 텍스트)나 정수, 부동소수점, binary data(비디오, 이미지, 오디오 파일)와 같이 어떠한 종류의 데이터라도 저장할 수 있다.</br>
문자열의 값의 데이터 용량은 512MB를 초과할 수 없다.</br>
개수 계산(count) 기능이 있다. (특정 명령을 통해 값을 +, - 하는 것을 말함)</br>
아래 예제에서 언급한 커맨드는 원자적(atomic) 커맨드로서, key-value 을 증가시키거나 감소시키고 하나의 명령으로 새 값을 리턴하는 커맨드 이다.</br>
즉, 서로 다른 두 개의 클라이언트가 동시에 하나의 키 값에 커맨드를 실행해도 동일한 값을 얻을 수 없다.</br>
커맨드 간의 어떠한 race condition(경합 조건)도 존재하지 않기 때문이다.</br>

> 원자적 : 다중 클라이언트가 동시에 동일한 키를 작업하는 커맨드를 수행할 때, race condition 이 결코 발생하지 않는 다는 것을 의미

**예제**

```
// MSET 은 한 번에 다중 key 에 value 를 설정할 수 있다.
127.0.0.1:6379> MSET first "First Key Value" second "Second Key Value"
OK

// MGET 은 한번여 다중 key 에 value 를 볼 수 있다.
127.0.0.1:6379> mGET first second
1) "First Key Value"
2) "Second Key Value"

// EXPIRE 은 key 에 만료 시간을 설정할 수 있다.
// TTL(Time To Live : 키 생존시간) 는 양의 정수, -2, -1 값을 리턴
127.0.0.1:6379> SET current_chapter "Chapter 1" // 특정 키 에 값을 설정
OK
127.0.0.1:6379> EXPIRE current_chapter 10 // 특정 키 값에 만료 시간 설정
(integer) 1
127.0.0.1:6379> TTL current_chapter // 만료시간이 설정되어 있으면, TTL 시간이 초로 계산되어서 나온다.
(integer) 7
...
...
...
127.0.0.1:6379> TTL current_chapter
(integer) 2
127.0.0.1:6379> TTL current_chapter
(integer) 1
127.0.0.1:6379> TTL current_chapter
(integer) 0
127.0.0.1:6379> TTL current_chapter // 키가 만료되거나 존재하지 않으면 -2 을 리턴한다.
(integer) -2
127.0.0.1:6379> SET current_chapter "Chapter 1"
OK
127.0.0.1:6379> TTL current_chapter // 키가 존재하지만 만료시간을 설정하지 않았다면 -1 을 리턴한다.
(integer) -1

// INCR 는 하나씩 키 값을 증가
// INCRBY 는 주어진 숫자 만큼 키 값을 증가
// DECR 과 DECRBY 는 INCR, INCRBY 와 반대로 키의 값을 감소
// INCRBYFLOAT 는 부동소수점을 받아 키 값을 증가시킨 후 새롭게 변경된 값을 리턴
127.0.0.1:6379> SET counter 10
OK
127.0.0.1:6379> INCR counter
(integer) 11
127.0.0.1:6379> INCR counter
(integer) 12
127.0.0.1:6379> INCRBY counter 5
(integer) 17
127.0.0.1:6379> DECR counter
(integer) 16
127.0.0.1:6379> DECRBY counter 5
(integer) 11
127.0.0.1:6379> GET counter
"11"
127.0.0.1:6379> INCRBYFLOAT counter 2.4
"13.4"
```

### 리스트

리스트는 간단한 collection, stack, queue와 같이 동작할 수 있기 때문에 레디스에서는 매우 유연한 데이터 타입이다.</br>
리스트 커멘드가 **원자적인 특성을 갖고 있어, 병렬 시스템이 큐에서 엘리멘트를 얻어낼 때 중복으로 얻지 않도록 보장**하고 있다.</br>
또, **blocking 커멘드가 존재**한다. 즉, 클라이언트가 비어있는 리스트에 블로킹 커멘드를 실행할 때, 클라이언트는 리스트에 새로운 엘리먼트를 추가될 때까지 기다린다는 의미다.</br>
자료구조는 연결리스트(linked list)로 되어 있어, 처음과 끝에서의 엘리먼트의 추가 및 삭제는 항상 O(1), 일정 시간의 성능을 가진다.</br>
look up 연산은 O(N), 선형(linear) 시간이 소요된다.</br>
가질 수 있는 최대 엘리먼트의 개수는 $2^{32}-1$ 이며, 40억 개 이상의 엘리먼트를 가질 수 있다는 것을 의미한다.</br>

특징

- 원자적인 특성
- blocking command
- linked list 자료구조 사용
- 최대 엘리먼트의 개수는 $2^{32}-1$

사용 사례

- 이벤트 큐

예제

```
// LPUSH : 리스트 처음에 데이터 추가 , return : 리스트의 길이
// RPUSH : 리스트 마지막에 데이터 추가 , return : 리스트의 길이
127.0.0.1:6379> LPUSH books "Clean Code"
(integer) 1
127.0.0.1:6379> RPUSH books "Code Complete"
(integer) 2
127.0.0.1:6379> LPUSH books "Peopleware"
(integer) 3

// LLEN : 리스트 길이
// LINDEX : 특정 index 의 value 를 return
127.0.0.1:6379> LLEN books
(integer) 3
127.0.0.1:6379> LINDEX books 1
"Clean Code"

// LPOP : 리스트의 첫 번째 엘리먼트를 삭제하고 리턴
// RPOP : 리스트의 마지막 엘리먼트를 삭제하고 리턴
// LRANGE : x 인덱스 에서 y index 까지 출력
127.0.0.1:6379> LPOP books
"Peopleware"
127.0.0.1:6379> RPOP books
"Code Complete"
127.0.0.1:6379> LRANGE books 0 -1
1) "Clean Code"
127.0.0.1:6379> LRANGE books 0 3
1) "Peopleware"
2) "Clean Code"
3) "Clean Code"
4) "Code Complete"
```

### 해시

해시에서 필드 이름과 값은 문자열이다. 따라서 해시는 문자열을 문자열로 매핑한다.</br>
해시의 큰 장점은 메모리 최적화이다. 최적화는 hash-max-ziplist-entries, hash-max-ziplist-value  설정을 기반으로 한다. (4장에서 추가 설명)</br>
해시는 내부적으로 ziplist, hash table 이 될 수 있다.</br>

- ziplist
    - 메모리 효율화에 목적을 둔 양쪽으로 연결된 연결리스트
    - 정수를 일련의 문자열로 저장하지 않고 실제로 정수의 값으로 저장
    - 메모리에 최적화 되어있다고 해서 일정한 시간 내로 검색이 수행되지 않는다.
- hash table
    - 일정한 시간내로 검색은 가능하지만, 메모리 최적화가 이루어지지 않는다.

관련 자료 : [https://instagram-engineering.com/storing-hundreds-of-millions-of-simple-key-value-pairs-in-redis-1091ae80f74c](https://instagram-engineering.com/storing-hundreds-of-millions-of-simple-key-value-pairs-in-redis-1091ae80f74c)

예제

```
// HSET : 주어진 키에 값 추가
// HMSET : 공백으로 구분하여 다중 필드 값을 키에 저장
// HINCRBY : 주어진 정수만큼 필드의 값을 증가
127.0.0.1:6379> HSET movie "title" "The Godfather"
(integer) 1
127.0.0.1:6379> HMSET movie "year" 1972 "rating" 9.2 "watchers" 1000000
OK
127.0.0.1:6379> HINCRBY movie "watchers" 3 // value 가 string 인 경우 에러 발생((error) ERR hash value is not an integer)
(integer) 1000003

// HGET : 해당 키 값을 조회
// HMGET : 다중 키 값을 조회
127.0.0.1:6379> HGET movie "title"
"The Godfather"
127.0.0.1:6379> HMGET movie "title" "watchers"

// HDEL : 해당 키 값을 삭제
127.0.0.1:6379> HDEL movie "watchers"
(integer) 1

// HGETALL : 모든 key, value 쌍으로 이루어진 배열을 리턴
127.0.0.1:6379> HGETALL movie
1) "title"
2) "The Godfather"
3) "year"
4) "1972"
5) "rating"
6) "9.2"
```

## 2. 고급 데이터 타입

### 셋(set)

레디스에서의 셋은 순서가 없고 동일한 문자열이 없는 컬렉션이다.</br>
내부적으로 set 은 hash table 로 구현 되어 있다.</br>
셋이 가질 수 있는 엘리먼트의 최대 개수는 $2^{32}-1$ 이다. 즉, 한 셋에 40억개 이상의 엘리먼트를 저장할 수 있다.</br>

예시

- 데이터 필터링 : 예를 들어, 주어진 한 도시에서 출발해서 다른 도시로 도착하는 모든 비행기를 필터링하기
- 데이터 그룹핑 : 비슷한 제품을 보는 모든 사용자를 그룹핑하기
- 엘리먼트십 확인 : 사용자가 블랙리스트에 있는지 확인하기

예제

```
// 예제는 음악 어플리케이션을 기반으로하며 음악 어플리케이션의 모든 사용자는 선호하는 음악가 셋을 가진다.
// 예제에서는 한 사용자 계정에서 선호하는 가수를 추가 및 삭제하고, 두 사용자가 공통적으로 선호하는 가수를 찾으며, 다른 사용자의 선호도를 기반으로 가수를 탐색한다.

// SADD : 하나 이상의 엘리먼트를 추가
127.0.0.1:6379> SADD user:max:favorite_aritst "Arcade Fire" "Arctic Monkeys" "Belle & Sebastian" "Lenine"
(integer) 4
127.0.0.1:6379> SADD user:hugo:favorite_arists "Daft Punk" "The Kooks" "Arctic Monkeys"
(integer) 3

// SINTER : 하나 이상의 셋을 받아 공통으로 존재하는 엘리먼트를 배열로 리턴
127.0.0.1:6379> SINTER user:max:favorite_aritst user:hugo:favorite_arists
1) "Arctic Monkeys"

// SDIFF : 하나 이상의 셋을 메게변수로 받고, 첫번째 셋의 기준으로 교집합을 찾아 리턴 (메게변수의 순서가 중요)
127.0.0.1:6379> SDIFF user:max:favorite_aritst user:hugo:favorite_arists
1) "Belle & Sebastian"
2) "Lenine"
3) "Arcade Fire"

// SUNION : 하나 이상의 셋을 메게변수로 받고, 모든 셋의 엘리먼트를 하나의 배열로 받아 리턴
127.0.0.1:6379> SUNION user:max:favorite_aritst user:hugo:favorite_arists
1) "Arcade Fire"
2) "Daft Punk"
3) "Arctic Monkeys"
4) "Belle & Sebastian"
5) "The Kooks"
6) "Lenine"

// SRANDMEMBER : 셋에서 무작위로 하나의 엘리먼트를 추출
127.0.0.1:6379> SRANDMEMBER user:max:favorite_aritst
"Belle & Sebastian"
127.0.0.1:6379> SRANDMEMBER user:max:favorite_aritst
"Lenine"

// SMEMBERS : 모든 엘리먼트를 배열로 추출
127.0.0.1:6379> SMEMBERS user:max:favorite_aritst
1) "Belle & Sebastian"
2) "Lenine"
3) "Arctic Monkeys"
4) "Arcade Fire"
```

### 정렬된 셋

정렬된 셋은 셋과 비슷하지만, 정렬된 셋의 모든 엘리먼트는 연관 점수를 가진다.</br>
정렬된 셋은 정렬되어있으면서 중복 문자열이 없는 컬렉션이다.</br>
반복 점수를 가진 엘리먼트는 가질 수 있다.</br>
이런 경우에 반복 엘리먼트를 사전 편집 순서(알파벳 순서)대로 정렬할 수 있다.</br>
엘리먼트의 추가 및 삭제, 변경 성능은 O(log(N))이며, 여기서 N 은 정렬된 셋의 엘리먼트 개수다.</br>
정렬된 셋의 내부 구현은 2개의 분리된 데이터 구조로 되어 있다.</br>

- 해시 테이블이 존재하는 skip list가 있다. 이는 순서대로 정렬된 엘리먼트를 빠르게 검색할 수 있는 데이터 구조이다.
- zset-max-ziplist-entries 와 zset-max-ziplist-value 설정을 기반으로 하는 ziplist 다.

사용 예시

- 고객 서비스에서 실시간 대기 목록 만들기
- 상위 점수를 가진 사용자, 비슷한 점수를 가진 사용자, 친구의 점수를 보여주는 큰 규모의 온라인 게임에서 리더보드(leader board) 보여주기
- 수백만 개의 단어를 사용한 자동 완성 시스템 만들기

예제

```
// ZADD : 정렬된 셋에 하나 이상의 엘리먼트 추가
127.0.0.1:6379> ZADD leader 100 "Alice"
(integer) 1
127.0.0.1:6379> ZADD leader 100 "Zed"
(integer) 1
127.0.0.1:6379> ZADD leader 102 "Hugo"
(integer) 1
127.0.0.1:6379> ZADD leader 101 "Max"
(integer) 1

// ZREVRANGE : 높은 점수에서 낮은 점수로 엘리먼트 리턴, 시작과 종료 인덱스 입력 가능 (동점이면 사전 편집 순으로 내림차순 정렬)
127.0.0.1:6379> ZREVRANGE leader 0 -1
1) "Hugo"
2) "Max"
3) "Zed"
4) "Alice"

// ZRANGE : ZREVRANGE 와 반대
127.0.0.1:6379> ZRANGE leader 0 -1
1) "Alice"
2) "Zed"
3) "Max"
4) "Hugo"

// WITHSCORES 옵션 : 엘리먼트에 점수와 함께 리턴
127.0.0.1:6379> ZRANGE leader 0 -1 WITHSCORES
1) "Alice"
2) "100"
3) "Zed"
4) "100"
5) "Max"
6) "101"
7) "Hugo"
8) "102"

// ZREM : 엘리먼트 삭제
127.0.0.1:6379> ZREM leader "Hugo"
(integer) 1

// ZSCORE : 엘리먼트의 점수 리턴
127.0.0.1:6379> ZSCORE leader "Max"
"101"

// ZRANK : 등수가 낮은 순에서 높은 순으로 정렬된 엘리먼트
// ZREVRANK : 등수가 높은 순에서 낮은 순으로 정렬된 엘리먼트
127.0.0.1:6379> ZRANK leader "Max"
(integer) 2
127.0.0.1:6379> ZREVRANK leader "Max"
(integer) 0
```

### 비트맵

비트맵은 레디스의 실제 데이터 타입이 아니다. 내부적으로는 문자열이다.</br>
비트맵은 문자열의 bit 연산자 집합이라고 보면 된다.</br>
비트맵을 0과 1로 구성된 배열로 생각할 수 있는데, 메모리 효율이 좋고 빠르고 데이터 검색을 지원하며 $2^{32}$ 비트(40억 비트 이상)까지 저장할 수 있다.</br>

사용 예시

- 각 유저별 행동을 2가지 조건(true, false) 으로 구분할 때

예제

```
// 각 비트맵 오프셋은 사용자를 표현하는데, 사용자 1은 오프셋, 사용자 30은 오프셋 30을 나타내며 나머지는 동일
// 비트맵이 10011 이라고 가정하면, 사용자 (0,3,4) 는 1로 표시돼 있기 때문에 웹사이트를 방문했다는 의미

// SETBIT : 비트맵 오프셋에 값을 저장하기 위해 사용하며 값은 1, 0 만 입력받는다.
127.0.0.1:6379> SETBIT visits:2015-01-01 10 1
(integer) 0
127.0.0.1:6379> SETBIT visits:2015-01-01 15 1
(integer) 0
127.0.0.1:6379> SETBIT visits:2015-01-02 10 1
(integer) 0
127.0.0.1:6379> SETBIT visits:2015-01-02 11 1
(integer) 0

// GETBIT : 비트맵 오프셋 값을 리턴
127.0.0.1:6379> GETBIT visits:2015-01-01 10
(integer) 1
127.0.0.1:6379> GETBIT visits:2015-01-02 15
(integer) 0

// BITCOUNT : 비트맵에 1로 표시된 모든 비트의 개수를 리턴
127.0.0.1:6379> BITCOUNT visits:2015-01-01
(integer) 2
127.0.0.1:6379> BITCOUNT visits:2015-01-02
(integer) 2

// BITOP : 옵션 (OR, AND, XOR, NOT) 이 있다. 비트맵을 변수로 받아 하나의 변수를 생성한다.
127.0.0.1:6379> BITOP OR total_users visits:2015-01-01 visits:2015-01-02
(integer) 2
127.0.0.1:6379> BITCOUNT total_users
(integer) 3
```

### 하이퍼로그로그(HyperLogLog)

HyperLogLog 는 레디스의 실제 데이터 타입이 아니다. 개념적으로는 알고리즘이다.</br>
셋에 존재하는 고유 엘리먼트 개수를 아주 좋은 근사치로 제공하기 위해 확률화를 사용하는 알고리즘 이다.</br>
하나의 키당 아주 작은 메모리(최대 12KB의 메모리)를 사용하며, 항상 O(1) 로 동작한다.</br>
레디스는 HyperLogLog 알고리즘을 사용해 셋의 개수를 계산하기 위한 문자열을 제어하는 커멘드를 제공한다. 그렇기 때문에 데이터 타입처럼 다룬다.</br>
HyperLogLog 알고리즘은 100%의 정확도를 보장하지 않는 확률적인 알고리즘이며 100% 정확하지 않고 0.81%의 표준오차를 가진다. (정확도를 계산하면 99.19% 이다)</br>

보통 종종 고유 개수를 계산하기 위해, 현재 계산 중인 셋의 엘리먼트 개수에 비례하는 메모리가 있어야 한다.</br>
HyperLogLog는 좋은 성능, 낮은 계산 비용, 적은 메모리양으로 해당 문제를 해결한다.</br>

예시

- 웹사이트의 고유 방문자 수 계산
- 특정 날짜 또는 시간에 웹사이트에서 검색한 고유 키워드 개수 계산
- 사용자가 사용한 고유 해시태그 개수 계산
- 책에 나오는 고유 단어 개수 계산

고유 방문자 수 계산 : HyperLogLog vs Set

<img src="/Redis-Essentials/blob/master/img/img-1.png" width="500px;">

예제

```
// 방문한 고유 방문자 수를 저장하고 계산하는 하이퍼로그로그 커멘드

// PFADD : 하나 이상의 문자열을 하이퍼로그로그에 추가 (개수가 변경되면 1, 그렇지 않으면 0)
127.0.0.1:6379> PFADD visits:2015-01-01 "carl" "max" "hugo" "arthur"
(integer) 1
127.0.0.1:6379> PFADD visits:2015-01-01 "max" "hugo"
(integer) 0
127.0.0.1:6379> PFADD visits:2015-01-02 "max" "kc" "hugo" "renta"
(integer) 1

// PFCOUNT : 근사치 개수를 리턴
127.0.0.1:6379> PFCOUNT visits:2015-01-01
(integer) 4
127.0.0.1:6379> PFCOUNT visits:2015-01-02
(integer) 4

// PFMERGE : 명세한 모든 하이퍼로그로그를 병합하고, 병합한 결과를 대상 키에 저장
127.0.0.1:6379> PFMERGE visits:total visits:2015-01-01 visits:2015-01-02
OK
127.0.0.1:6379> PFCOUNT visits:total
(integer) 6
```

## 3. 시계열

필요할 때 다시보기

## 4. 커맨드

### Pub/Sub

Pub/Sub 은 메시지를 특정 수신자에게 직접 발송하지 못하는 곳에서 쓰이는 패턴 Publish-Subscript (발송하다-구독하다) 의 약자다.</br>
발송자(publisher)와 구독자(subscriber)가 특정 채널을 리스닝하고 있다면, 발송자는 채널에 메시지를 보내고 구독자는 발송자의 메시지를 받는다.</br>
레디스는 Pub/Sub 패턴을 지원하고, 메시지를 발송하고 채널을 구독하는 커맨드를 제공한다.</br>

사용 예시

- 뉴스와 날씨 대시보드
- 채팅 어플리케이션
- 지하철 지연 경고 등의 푸시 알림

예제

```
// SUBSCRIBE : 클라이언트가 하나 이상의 채널을 구독
// UNSUBSCRIBE : 하나 이상의 채널에서 클라이언트를 구독 해지
// PUBSUB : Pub/Sub 시스템의 상태를 조사

* 레디스 클라이언트가 SUBSCRIBE 커멘드를 실행하게 되면 구독 모드로 들어가기 때문에 SUBSCRIBE, UNSUBSCRIBE 커멘더 외에는 받지 않는다.
```

### **트랜잭션**

레디스의 트랜잭션은 순서대로 원자적으로 실행되는 커맨드 열이다.</br>
MULTI 커멘드는 트랜잭션의 시작을  표시하고 EXEC 커멘드는 트랜잭션 마지막을 표시한다.</br>
MULTI 와 EXEC 커맨드 간의 모든 커멘드는 직렬화 되며, 원자적으로 실행되며 트랜잭션이 실행될 동안에는 다른 클라이언트의 요청을 받지 않는다.</br>
트랜잭션의 모든 커멘드는 클라이언트의 큐에 쌓이고, EXEC 커멘드가 실행되면 서버로 바로 전달된다.</br>

**단점**

기존의 SQL 데이터베이스와 달리, 레디스의 트랜잭션은 트랜잭션 과정을 실행하다 실패하더라도 롤백하지 않는다.</br>
레디스는 트랜잭션의 커멘드를 순서대로 실행하고, 커멘드 중 일부가 실패하면 다음 커멘드로 처리한다.</br>
또, 트랜잭션 수행중에는 모든 커멘드가 큐에 쌓이기 때문에 트랜잭션 내부에서 어떠한 결정도 내릴 수 없다.</br>

### 파이프라인

파이프라인은 다중 커멘드를 레디스 서버에 한꺼번에 보내는 방법을 말하며, 개별 응답을 기다리지 않고 클라이언트에 의해 응답을 한 번에 읽을 수 있다.</br>
개별 응답을 기다리지 않고 클라이언트에 의해 응답을 한 번에 읽을 수 있다.</br>
레디스 클라이언트가 커맨드를 보내고 레디스 서버로부터 응답을 받는데 걸리는 시간을 RTT(Round Tirp Time) 라 하는데, 다중 커멘드를 레디스로 보내면 다중 RTT 가 생긴다.</br>
그렇지만 파이프라인을 사용하게 되면 커멘드를 그룹화 하기 때문에 RTT 값을 줄일 수 있어, 열 개의 커맨드를 파이프라인으로 이용하면 하나의 RTT 가 된다. 파이프라인은 레디스의 네트워크 성능을 확연히 개선할 수 있다.</br>
예를 들면, 종종 부하가 높은 애플리케이션이 네트워크 병목 현상을 발생시키는데, 파이프라인은 네트워크 병목이 발생하지 않게해 네트워크 시간을 많이 줄일 수 있기 때문에 유용하다.</br>

파이프라인으로 전달된 레디스 커멘드는 독립적이여야 한다. 이유는 순차적으로 실행은 되지만 트랜잭션 처리가 되지 않는다.</br>
또한, 트랜잭션의 커멘드는 기본적으로 파이프라인으로는 보내지 않는다. 그래서 파이프라인에 트랜잭션을 전달하는 것도 유용한 방법이다.</br>

### 기타 커멘드

**INFO**

INFO 커멘드는 레디스 버전과 운영체제, 연결된 클라이언트, 메모리 사용량, 저장소, 복제본, keyspace 에 대한 정보를 포함한 모든 레디스 서버 통계를 리턴한다.</br>
매개변수로 섹션 이름을 명세해 결과를 제한할 수도 있다.</br>

예제

```java
127.0.0.1:6379> info memory
# Memory
used_memory:864112
used_memory_human:843.86K
used_memory_rss:7774208
used_memory_rss_human:7.41M
used_memory_peak:922184
used_memory_peak_human:900.57K
used_memory_peak_perc:93.70%
used_memory_overhead:822632
used_memory_startup:802128
used_memory_dataset:41480
used_memory_dataset_perc:66.92%
allocator_allocated:1041880
allocator_active:1347584
allocator_resident:4161536
total_system_memory:8348213248
total_system_memory_human:7.77G
used_memory_lua:37888
used_memory_lua_human:37.00K
used_memory_scripts:0
used_memory_scripts_human:0B
number_of_cached_scripts:0
maxmemory:0
maxmemory_human:0B
maxmemory_policy:noeviction
allocator_frag_ratio:1.29
allocator_frag_bytes:305704
allocator_rss_ratio:3.09
allocator_rss_bytes:2813952
rss_overhead_ratio:1.87
rss_overhead_bytes:3612672
mem_fragmentation_ratio:9.45
mem_fragmentation_bytes:6951112
mem_not_counted_for_evict:0
mem_replication_backlog:0
mem_clients_slaves:0
mem_clients_normal:20504
mem_aof_buffer:0
mem_allocator:jemalloc-5.1.0
active_defrag_running:0
lazyfree_pending_objects:0
```

**DBSIZE**

레디스 서버에 존재하는 키 개수를 리턴한다

**DEBUG SEGFAULT**

올바르지 않은 메모리 접근을 수행해 레디스 서버 프로세스를 종료한다.</br>
애플리케이션 개발 중에 버그를 시뮬레이션할 때 유용하다.</br>

**MONITOR**

레디스 서버가 처리하는 모든 커멘드를 실시간 으로 보여준다.</br>
주의할점은, MONITOR 가 레디스 성능을 50% 이상까지 떨어뜨릴 수 있다고 한다.(공식문서)</br>

**CLIENT LIST 와 CLIENT SETNAME 커맨드**

CLIENT LIST 커맨드는 클라이언트에 대한 관련 정보와 통계뿐 아니라 서버에 연결된 모든 클라이언트 목록을 리턴한다.(IP, idle time, 등)</br>
CLIENT SETNAME 커맨드는 클라이언트 이름을 변경한다. 디버깅 목적으로 할 때 유용하다.</br>

**CLIENT KILL**

클라이언트 연결을 종료한다.</br>
IP나 포트, ID, 타입 으로 클라이언트 연결을 종료할 수 있다.</br>

예제

```java
CLIENT KILL ADDR 127.0.0.1:51167
CLIENT KILL ID 22
CLIENT KILL TYPE slave
```
**FLUSHALL**

레디스의 모든 키를 삭제한다.</br>
삭제된 키는 복구가 불가능하다.</br>

**RANDOMKEY**

존재하는 키 이름 중 무작위로 선택한 하나의 키 이름을 리턴한다.</br>
사용중인 키를 대략적으로 보고 싶을 때 사용하면 된다.</br>
(KEYS 는 모든 key 를 분석하기 때문에 리소스 소모가 좀 더 크다)</br>

**EXPIRE 와 EXPIREAT**

EXPIRE 커맨드는 특정 키의 타임아웃을 초 단위로 설정할 수 있다.</br>
음수의 타임아웃은 키를 바로 삭제한다. (DEL 과 같다고 보면 된다.)</br>
EXPIREAT 커맨드는 유닉스 타임스탬프를 기반으로 특정 키의 타임아웃을 설정한다.</br>
설정한 타임스탬프가 과거라면 즉시 삭제된다.</br>

예제

```java
127.0.0.1:6379> MSET key1 value1 key2 value2
OK
127.0.0.1:6379> expire key1 30
(integer) 1
127.0.0.1:6379> expireat key2 1435717600
(integer) 1
```

**TTL 과 PTTL**

TTL 커맨드는 타임아웃 값이 있는 키의 남아있는 생존 시간을 초 단위로 리턴한다.</br>
PTTL 커맨드는 남아 있는 생존시간을 밀리 초 단위로 리턴한다.</br>

**PERSIST**

특정 키에 주어진 현존 타임아웃을 제거한다.</br>
새로운 타임아웃을 설정하지 않으면, 키는 만료되지 않는다.</br>

**SETEX**

특정 키에 값을 저장할 때, 만료 시간도 함께 원자적으로 설정한다.

예제

```java
127.0.0.1:6379> SETEX mykey 30 value
OK
127.0.0.1:6379> get mykey
"value"
127.0.0.1:6379> ttl mykey
(integer) 21
```

**DEL**

하나 이상의 키를 레디스에서 삭제하고 삭제된 키의 개수를 반환한다.

예제

```java
127.0.0.1:6379> MSET key1 value1 key2 value2
OK
127.0.0.1:6379> del key1 key2
(integer) 2
```

**EXISTS**

특정 키가 존재하면 1 을 리턴하고, 키가 존재하지 않다면 0 을 리턴한다.

**PING**

서버/클라이언트의 연결을 테스트하고,  레디스가 데이터를 교환할 수 있는 상태를 확인할 때 유용하다.

**MIGRATE**

특정 키를 대상 레디스 서버로 옮긴다.</br>
MIGRATE 는 원자적이므로 키를 옮기는 동안 원본 레디스 서버와 키를 저장할 레디스 서버가 둘다 블록된다.</br>
키를 저장할 레디스 서버에 이미 키가 존재한다면 MIGRATE 커맨드는 실패한다.</br>

예제

```java
MIGRATE host port key destination-db timeout [COPY] [REPACE]

// 2개의 매개변수가 있다.
// COPY : 로컬 레디스 서버에 키를 유지하고 키를 저장할 레디스 서버에 키의 복사본을 생성
// REPLCAE : 키를 저장할 레디스 서버에 이미 동일한 키가 존재하더라도 키를 교체한다.
```

**AUTH**

레디스에 연결할 수 있는 클라이언트를 authorization 하는데 사용된다.

**SHUTDOWN**

모든 클라이언트를 종료하고, 최대한 데이터를 저장하려고 한 후, 레디스 서버를 종료한다.

예제

```java
127.0.0.1:6379> SHUTDOWN SAVE
127.0.0.1:6379> SHUTDOWN NOSAVE

// 2개의 매개변수가 있다.
// SAVE : persistence 기능을 활성화 하지 않더라도 레디스가 dump.rdb 라는 파일에 모든 데이터를 저장하도록 강제
// NOSAVE : persistence 기능이 활성화 되어 있더라도, 레디스 서버가 데이터를 디스크에 저장하지 않도록 한다.
```

**OBJECT ENCODING**

주어진 키에서 사용 중인 인코딩 값을 리턴한다.

예제

```java
127.0.0.1:6379> HSET myhash field value
(integer) 1
127.0.0.1:6379> object encoding myhash
"ziplist"
```

### 데이터 타입 최적화

레디스에서 모든 데이터 타입은 메모리를 저장하거나 성능을 높이는 다양한 인코딩을 사용할 수 있다.</br>
데이터 타입은 레디스의 서버 설정에 정의된 임계 값을 기반으로 다양한 인코딩을 사용한다.</br>
보통 redis.conf 파일의 기본 값은 대부분의 애플리케이션에서 사용하기에 충분하다.</br>
또한, CONFIG 커맨드나 커맨드 라인의 옵션으로 레디스 설정을 명세할 수 있다. (대부분의 방식은 설정 파일을 사용하는 것이다.)</br>

**문자열**

int : 64비트 부호 있는 정수로 문자열을 표현할 때 사용된다.</br>
embstr : 40바이트 보다 작은 문자열을 표현할 때 사용된다.</br>
raw : 40바이트보다 큰 문자열을 표현할 때 사용된다.</br>

예제

```
127.0.0.1:6379> set str1 12345
OK
127.0.0.1:6379> object encoding str1
"int"
127.0.0.1:6379> set str2 "An embstr is small"
OK
127.0.0.1:6379> object encoding str2
"embstr"
127.0.0.1:6379> set str3 "A raw enconded String is anthing greater than 39 bytes"
OK
127.0.0.1:6379> object encoding str3
"raw"
```

**리스트 (redis 버전별로 다른것 같음)**

ziplist : 리스트의 크기의 엘리먼트가 list-max-ziplist-entries 설정보다 작고, 리스트의 개별 엘리먼트의 바이트가 list-max-ziplist-value 설정보다 작다면 사용된다.</br>
linked-list : ziplist 에 해당되지 않으면 linked list 가 적용된다.</br>

**셋**

intset : 셋의 모든 엘리먼트가 정수이며, 셋의 개수가 set-max-intset-entries 설정보다 작으면 사용된다.</br>
hashtable : 셋의 엘리먼트 중 하나라도 정수가 아니거나, 셋의 개수가 set-max-intset-entries 설정보다 크면 사용된다.</br>

예제

```
127.0.0.1:6379> sadd set1 1 2
(integer) 2
127.0.0.1:6379> object encoding set1
"intset"
127.0.0.1:6379> sadd set2 1 2 3 4 5
(integer) 5
127.0.0.1:6379> object encoding set2
"intset"
127.0.0.1:6379> sadd set3 a
(integer) 1
127.0.0.1:6379> object encoding set3
"hashtable"
```

**해시**

ziplist : 해시의 필드 개수가 hash-max-ziplist-entries 설정보다 작고, 해시의 필드 이름과 값이 hash-max-ziplist-value 설정보다 작으면 사용된다.</br>
hashtable : ziplist 에 해당되지 않으면 사용된다.</br>

**정렬된 셋**

ziplist : 정렬된 셋의 개수가 set-max-ziplist-entries 설정보다 작고, 정렬된 셋의 엘리먼트 값이 모두 zset-max-ziplist-value 설정보다 작으면 사용된다.</br>
skiplist : ziplsit 에 해당되지 않으면 사용된다.</br>

## 6. 일반적인 실수(실수 피하기)

### 다중 레디스 데이터베이스

레디스는 단일 스레드 기반이므로, 하나의 레디스 서버의 다중 데이터베이스는 하나의 CPU 코어만 사용한다.</br>
반면, 동일 장비에 여러 개의 레디스 서버를 실행하면, 다중 CPU 코어의 장점을 취할 수 있다. (하지만 서비스의 용도, 구성에 따라 적절하게 선택하는게 필요하다.)</br>

## 7. 보안 기술(데이터 보호하기)

### 기술적인 보안

레디스는 Access Control List(ACL)을 구현하지 않았다. 따라서 퍼미션 레벨 단위로 사용자를 구분할 수 없다.</br>
다만, 인증 기능이 존재하는데 인증 기능은 requirepass 설정으로 활성화할 수 있다.</br>
레디스는 엄청 빠르기 때문에, 악성 유저가 초당 수천 개의 패스워드를 요청해 잠재적으로 알아 맞출 수 있어서 requirepass 설정 또한 위험할수 있다.</br>
따라서 최소 64 자의 복잡한 패스워드를 명세해 패스워드를 알아맞히지 못하게 해야 한다.</br>

### 네트워크 보안(레디스가 구동되고 있는 서버에 보안)

- 알 수 없는 클라이언트의 접속을 막을 수 있도록 방화벽 규칙을 적용
- 공개된 클라우드에서 네트워크 인터페이스로 바로 접속하지 못하게 하고, loopback 인터페이스로 레디스를 실행
- 공용 인터넷 대신 가상 사설 클라우드에서 레디스를 실행
- 클라이언트와 서버 간의 통신을 암호화

## 8. 레디스 확장하기(싱글 인스턴스 넘어서기)

## 저장

데이터 손실 문제를 해결하기 위해 레디스는 저장 가능한 Redis Database(RDB) , Append-only File(AOF)을 제공한다.</br>
동일한 레디스 인스턴스에서 두 방법을 개별적으로 또는 동시에 사용 가능하다.</br>

**Redis Database (RDB)**

.rdb 파일은 특정 시점의 데이터 바이너리로, 레디스 인스턴스에 저장된 데이터를 표현한다.</br>
RDB 파일 형태는 빠른 읽기와 쓰기에 최적화되어 있다.</br>
.rdb 파일의 내부 구조는 레디스 인메모리 구조와 매우 비슷하게 되어있어서 빠른 읽기로 복구가 가능하다.</br>
필요할때마다 매시간, 매일, 매주, 매달 RDB 파일을 저장할 수 있기 때문에 RDB 의 백업과 복구 기능은 훌륭하다.</br>
SAVE 커맨드는 RDB 파일을 즉시 생성하지만, snapshot 과정에서 레디스 서버를 블록하기 때문에 SAVE 커맨드를 사용하지 않도록 한다.</br>
대신에 BGSAVE 커맨드를 사용하면 블록되는 문제점을 해소할 수 있다.</br>

백그라운드로 RDB 파일을 저장할 때, 성능 저하가 발생하지 않도록 redis-server 는 모든 저장 작업을 수생할 자식 프로세스를 생성(fork) 한다.</br>
따라서 redis-server 프로세스는 어떠한 디스크 I/O 작업도 수행하지 않는다.</br>
redis-server 가 쓰기 작업을 받고 있는 상태면, 자식 프로세스는 변경된 메모리 페이지를 복사해야 한다.</br>
복사된 메모리로 인해 전체 사용 메모리가 현격하게 증가할 수 있다.</br>

예제

```java
-- redis.conf --

save 900 1 // 적어도 하나의 쓰기 작업이 실행되면, 900초마다 디스크 .rdb 파일을 저장
save 300 10 // 적어도 10개의 쓰기 작업이 실행되면, 300초마다 디스크 .rdb 파일을 저장
save 60 10000 // 적어도 10000개의 쓰기 작업이 실행되면, 60초마다 디스크 .rdb 파일을 저장

(설정 파일에 설정하지 않으면 디스크에 아무것도 저장하지 않는다.)

옵션
stop-writes-on-bgsave-error : "yes" or "no" // 해당 옵션은 백그라운드 저장이 실패했을때, 레디스가 쓰기 작업 받는 것을 멈추도록 결정하는 값이다. (default: yes)
rdbcompression : "yes" or "no" // .rdb 파일에 LZF 압축 사용 여부 (default : yes)
rdbchecksum : "yes" or "no" // .rdb 파일에 checksum 을 저장하고, .rdb 파일을 로딩하기 전에 체크섬을 수행할지에 대한 여부. 만약 .rdb 와 checksum 이 같지 않으면 redis 는 실행되지 않는다.(default : yes)
dbfilename : "filename" // .rdb 파일 이름 설정 (default : dump.rdb)
save : // 초와 변경 수를 기반으로 shapshot 주기 설정
dir : // AOF, RDB 파일의 디렉토리 위치 지정
```
