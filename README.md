# Concurrency Issue With Stock System
## 재고시스템으로 알아보는 동시성 이슈 해결방법

> 인프런 [Inflearn - 재고시스템으로 알아보는 동시성 이슈 해결방법](https://www.inflearn.com/course/%EB%8F%99%EC%8B%9C%EC%84%B1%EC%9D%B4%EC%8A%88-%EC%9E%AC%EA%B3%A0%EC%8B%9C%EC%8A%A4%ED%85%9C) 강의를 수강하였습니다.

## 문제발생 케이스
하나의 상품이 100개의 재고를 가지고 있을 때 단순하게 생각하면 판매될 때 재고를 -1 한 후 저장해주면 될거라고 생각할 수 있지만, 실무에서는 다양한 케이스가 발생할 수 있다.

대표적으로,
하나의 상품에 주문이 동시에 (ms 차이 수준)  들어온다고 가정을 한다.
이럴 때, Race Condition이 발생하여 마지막 재고 값이 이상하게 나올 수 있다.

100개의 쓰레드가 같은 상품의 재고를 감소시켰을 때 최종 재고는 0을 기대하지만 실제 테스트는 실패하는 것을 확인 할 수 있다.

## 해결 방법
## 1. Syncronized 
Java의 기본 Syncronized 키워드를 사용해서 하나의 쓰레드씩 해당 메소드를 호출 할 수 있도록 수정한다.

예시처럼 코드를 작성하면 쓰레드 동기화를 통해 해결했다고 생각할 수 있지만, 결과는 실패한다.
이유는 `@Transactiional` Proxy 방식 때문에 Proxy가 (Commit)종료되기 전에 다른 쓰레드가 해당 자원에 접근할 수 있기 때문에 실제 데이터베이스의 값은 변경되지 않았을 수 있다.

### 해결법
서비스에 붙어있는 `@Transactional` 을 없앤다.
`@Transactional`을 제거하면 Proxy로 동작하지 않기 떄문에 해결을 할 수는 있다.

### Syncronized 문제점
Syncronized는 하나의 프로세스에서만 동기화를 보장한다.
현대의 어플리케이션은 2대 이상의 서버를 사용하기 때문에 Syncronized 로는 보장할 수 없다.

## 2. Mysql 을 활용한 다양한 방법 

### Pessimistic Lock을 활용
- 실제 데이터에 Lock을 걸어서 정합성을 맞추는 방법
- Exclusive Lock을 걸게되면 다른 트랜잭션에선느 Lock이 해제되기 전에는 데이터를 가져갈 수 없다.
- Dead Lock이 걸릴 수 있기 때문에 주의하기

```mysql
select stock0_.id as id1_0_, stock0_.product_id as product_2_0_, stock0_.quantity as quantity3_0_ from stock stock0_ where stock0_.id=? for update
```
`select..for update` 를 사용하여 해당 row를 다른 세션에서 건들 수 없도록 Lock을 건다.

##### 장점
- **충돌이 빈번하게 일어나면 Optimistic Lock보다 성능이 좋을 수 있음**
- Lock을 잡기 때문에 데이터 정합성을 지킬 수 있음
##### 단점
- 성능감소

### Optimistic Lock
- 실제 Lock을 이용하지 않고 버전을 이용함으로써 정합성을 맞추는 방법
- 먼저 데이터를 읽은 후 update를 수행할 때 현재 내가 읽은 버전이 맞는지 확인하여 업데이트
- 내가 읽은 버전에서 수정사항이 생겼을 경우 application 단에서 다시 읽은 후 작업을 수행해야 함

##### 장점
- 별도의 Lock이 없기 때문에 성능상 이점
##### 단점
- 실패했을 시 로직을 개발자가 직접 만들어야함
- 충돌이 빈번히 일어나면 성능상 손해

### Named Lock
- 이름을 가진 Metadata Locking (Table/Row Lock 이 아님)
- 이름을 가진 Lock을 획득한 후 해제할 때 까지 다른 세션은 이 Lock 을 획득할 수 없도록 합니다.
- Transaction이 종료될 때 자동으로 해제되지 않음
- 별도의 명령어로 해제/선점 시간이 끝나야 해제 Transaction이 종료될 때 Lock이 자동으로 해제되지 않음.

##### 특징
- JPA Native Query를 이용
- Connction Pool 이 부족해질 수 있기 때문에 별도의 Datasource를 이용하기를 추천


## 3. Redis 활용 
### 1. Lettuce
- `setnx` 명령어 사용
- spin lock 방식 
  - Lock을 획득하려는 스레드가 Lock을 획득할 수 있는지 반복적으로 확인하면서 Lock 획득 시도
  - retry 로직 필요

##### 장점
-   구현이 간단하다
-   spring data redis 를 이용하면 lettuce 가 기본이기때문에 별도의 라이브러리를 사용하지 않아도 된다.
##### 단점
- spin lock 방식이기때문에 동시에 많은 스레드가 lock 획득 대기 상태라면 redis 에 부하가 갈 수 있다.
    - Thread.sleep으로 부하를 줄여줬음

### 2. Redisson
- Pub-Sub 기반의 Lock 구현 제공
- 채널을 만들고 Lock을 획득한 Thread가 Lock획득을 시도하는 Thread에게 해제되었음을 알려주는 방식
##### 장점
-   락 획득 재시도를 기본으로 제공한다.
-   별도의 repository를 작성하지 않아도 된다.
-   pub-sub 방식으로 구현이 되어있기 때문에 lettuce 와 비교했을 때 redis 에 부하가 덜 간다.
##### 단점
-   별도의 라이브러리를 사용해야한다.
-   lock 을 라이브러리 차원에서 제공해주기 떄문에 사용법을 공부해야 한다.


### 실무에서는 ?
- 실패 시 재시도가 필요하지 않은 lock 은 lettuce 활용
- 실패 시 재시도가 필요한 경우에는 redisson 를 활용


## 추가 사항
📌 ExecutorService

- ExecutorService란, 병렬 작업 시 여러 개의 작업을 효율적으로 처리하기 위해 제공되는 JAVA 라이브러리다.
- ExecutorService는 손쉽게 ThreadPool을 구성하고 Task를 실행하고 관리할 수 있는 역할을 합니다.
- Executors 를 사용하여 ExecutorService 객체를 생성하며, 쓰레드 풀의 개수 및 종류를 지정할 수 있는 메소드를 제공한다.

📌 CountDownLatch

- CountDownLatch란, 어떤 스레드가 다른 쓰레드에서 작업이 완료될 때 가지 기다릴 수 있도록 해주는 클래스다.
- CountDownLatch 를 이용하여, 멀티스레드가 100번 작업이 모두 완료한 후, 테스트를 하도록 기다리게 한다.

## 참고 자료
[https://github.com/dev-alxndr/concurrency-stock](https://github.com/dev-alxndr/concurrency-stock)
[https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_exclusive_lock](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_exclusive_lock)   
[https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html)   
[https://dev.mysql.com/doc/refman/8.0/en/locking-functions.html](https://dev.mysql.com/doc/refman/8.0/en/locking-functions.html)
