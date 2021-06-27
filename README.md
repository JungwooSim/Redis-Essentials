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
