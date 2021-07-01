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
