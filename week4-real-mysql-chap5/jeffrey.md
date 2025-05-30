# 논의할 주제

> 데이터베이스 서버 1대에 5대의 웹 서버가 접속해서 서비스하는 상황에서 어떤 정보를 동기화해야하는 요건처럼 여러 클라이언트가 상호 동기화를 처리해야 할 때 네임드락을 사용하면 쉽게 해결할 수 있다.

1. 네임드 락을 사용해 쉽게 해결할 수 있는 구체적인 예시는 뭐가 있을까?

여러 대의 웹 서버가 주문번호 123을 획득하려고 할 때, 해당 123에 대한 락을 걸어 주문번호가 겹치지 않도록 잠그는 것.

> STATEMENT 포맷의 바이너리 로그를 사용하는 MySQL 서버에서는, REPEATABLE READ 격리 수준을 사용해야 한다. 또한 innodb_locks_unsafe_for_binlog 시스템 변수가
> 비활성화 되면 (0으로 설정되면) 변경을 위해 검색하는 레코드에는 넥스트 키 락 방식으로 잠금이 걸린다. InnoDB의 갭 락이나 넥스트 키 락은 바이너리 로그에 기록되는 쿼리가 레플리카 서버에서 실행될 때 소스
> 서버에서 만들어 낸 결과와 동일한 결과를 만들어내도록 보장하는 것이 주목적이다. 그런데 이외로 넥스트 키 락과 갭 락으로 인해 데드락이 발생하거나 다른 트랜잭션을 기다리게 만드는 일이 자주 발생한다. 가능하다면
> 바이너리 로그 포맷을 ROW 형태로 바꿔서 넥스트 키 락이나 갭 락을 줄이는 것이 좋다.

2. 바이너리 로그와, 격리수준과의 관계는 무엇일까? 왜 MySQL 서버는 바이너리 로그를 사용하기 때문에 해당 격리수준을 사용해야 하는걸까?

```
바이너리 로그 포맷
MySQL은 데이터 변경 사항을 기록하기 위해 **바이너리 로그(Binary Log)**를 사용합니다. 이 로그는 데이터 복제(Replication) 및 복구에 중요한 역할을 합니다. 바이너리 로그에는 세 가지 포맷이 있습니다:

STATEMENT: SQL 쿼리 자체를 기록합니다.

ROW: 변경된 데이터 행(row)만 기록합니다.

MIXED: STATEMENT와 ROW 방식을 혼합하여 사용합니다.

STATEMENT 포맷은 SQL 쿼리를 그대로 기록하기 때문에, 쿼리가 실행되는 환경에 따라 결과가 달라질 수 있습니다.
이를 방지하기 위해 REPEATABLE READ 격리 수준을 사용해야 함. 
why? -> Phantom Read는 하나의 트랜잭션에서 동일 쿼리를 실행할 시 다른 결과값이 나오는 부정합인  
```

---

트랜잭션과 락은 다른것

트랜잭션 -> 데이터의 정합성을 위해서
락 -> 데이터의 동시성을 위해서

결국 다양한 트랜잭션 종류와 락의 종류는, 결국 데이터의 정합성과 동시성을 위해서 이다.

# 트랜잭션과 잠금

트랜잭션과 잠금(락)은 서로 비슷한 개념 같지만,

잠금(락) : 동시성 제어가 목적 → 동시에 변경을 막기

트랜잭션 : 데이터 정합성이 목적 → 데이터들의 값이 서로 일치하도록

## 트랜잭션

MySQL에서의 트랜잭션

쿼리의 수와는 관련이 없고,

실행결과가 100 % 적용되거나 (Commit) 0% 적용되거나 (Rollback) 를 보장하는 것 .

트랜잭션이 없다면 ?

데이터 정합성을 맞추기 어려움 → 남은 레코드를 다시 삭제하는 재처리 작업이 필요하다.

주의사항

DB 커넥션을 가지고 있는 범위와, 트랜잭션이 활성화 되어있는 범위 최소화하기!

1. 트랜잭션은, 가능한 범위를 최소화하는게 좋다!
    1. 트랜잭션의 범위가 길어질수록, 각 단위프로그램이 데이터베이스 커넥션을 소유하는 시간이 길어져 , 커넥션 갯수에 유의, 자원을 많이 쓰게됨
2. 원격 서버통신 부분은 제거
    1. 서버통신이 불가능해지면, DB에도 영향이 감
    2. 네트워크 작업 배제하기
3. 꼭 필요한 부분 (연관된부분) 은 묶기
    1. 필요없는 조회같은 경우는 트랜잭션에서 제거

## MySQL 엔진의 잠금

락 : 트랜잭션들이 동시에 실행될 때, 일관성을 해치지 않도록 데이터 접근을 제어하는것

MySQL 엔진 잠금 : 전체에 영향을 미침

스토리지 엔진 잠금 : 해당 엔진에만 영향을 미침

종류

1. 글로벌 락
    1. MySQL 서버 전체에서, 글로벌 락을 획득한 세션 외에는 단순 Select만 가능
    2. 실행전 모든 테이블을 플러시해야하기 때문에, 기존 쿼리나 락이 완료되어야 함.
    3. 전체 테이블에 영향을 미치므로 가급적 사용X
    4. 이제는 잘 안씀. InnoDB 스토리지 엔진을 기본으로 채택하는데, 이 엔진은 트랜잭션을 지원함
    5. 트랜잭션을 위해 글로벌 락을 걸어 모든 테이블의 작업을 멈출 필요가 없어짐.
    6. ⇒ 백업락 도입
    7. 백업락? 글로벌 락에서, 일반적인 테이블 내의 데이터 변경을 허용한 락
    8. 서버는 소스 서버와 레플리카 서버로 나뉘고, 백업 실행은 주로 레플리카 서버에서 실행되는데, 백업에 글로벌 락을 걸면 소스 서버에 문제가 생기거나 DDL 명령 하나에도 백업에 실패할 수 있음
2. 테이블 락
    1. 명시적으로는 쓸일이 거의 없음
    2. 묵시적으로는, 쿼리 실행시 자동 확득 및 반납
    3. but 이것도 InnoDB 스토리지엔진에 레크드기반의 잠금을 지원하기 때문에 대부분의 데이터 변경 하는 DML 쿼리에는 무시되고, 스키마를 변경하는 쿼리 DDL 의 경우에만 영향을 미침
3. 네임드 락
    1. 테이블이나 레코드가 아니라, 사용자가 지정한 문자열에 대해 잠금
    2. 데이터베이스 서버 1대에 5대의 웹 서버가 접속해서 서비스하는 상황에서 어떤 정보를 동기화해야하는 요건처럼 여러 클라이언트가 상호 동기화를 처리해야 할 때 네임드락을 사용하면 쉽게 해결할 수 있다.
4. 메타데이터 락
    1. 테이블, 뷰 등의 객체 구조 변경, 명시적으로 선헌하는것이 아니라 이름 변경시 자동으로 획득하는 락.
    2. 테이블 의 구조를 변경해야 할 요건이 발생했다. 물론 MySQL 서버의 Online DDL을 이용해서 변경할 수도 있지만, 시간이 너무 오래 걸리는 경우라면 언두 로그의 증가와 Online DDL 이
       실행되는 동안 누적된 Online DDL 버퍼의 크기 등 고민해야 할 문제가 많다. 더 큰 문제는 MySQL 서버의 DDL 은 단일 스레드로 작동하기 때문에 상당히 많은 시간이 소모될 것이라는 점이다.

## InnoDB 스토리지 엔진 잠금

https://mangkyu.tistory.com/298

InnoDB 스토리지 엔진은 MySQL에서 제공하는 잠금과는 별개로 스토리지 엔진 내부에서 레코드 기반의 잠금 방식을 탑재하고 있다. 별도의 잠금방식을 가지고 있어, MySQL 명령으로 접근하기 어렵다.

이제는 대기중인 트랜잭션 목록을 조회할 수 있다.

InnoDB 스토리지 엔진의 잠금

레코드기반의 잠금 기능을 제공하며, 락 에스컬레이션이 일어나지 않는다.

락 에스컬레이션 : 좁은 범위의 다수의 락이 더 넓은 범위의 락으로 변하는 것.

→ 메모리 리소스를 더 효율적으로 사용하기 위해.

1. 레코드 락
    1. 레코드 자체만을 잠그는 것을 레코드 락.
    2. 테이블 레코드가 아닌 인덱스 레코드에 락을 건다 -> 인덱스 생성하지 않았는데 어디에?
    3. 테이블에 숨겨져 있는 clustered index를 사용하여 record 를 잠근다.
    4. InnoDB 스토리지 엔진은 레코드 자체가 아니라, 레코드의 인텍스를 잠금
2. 갭 락
    1. 레코드 사이의 간격에 새로운 레코드가 생성되는 것을 제어함.
    2. 레코드와 레코드 사이의 간격에 새로운 레코드가 생성되는 것을 제어하는 것.
    3. 넥스트 키 락의 일부로 주로 사용
    4. https://www.percona.com/blog/innodbs-gap-locks/
3. 넥스트 키 락
    1. 레코드락 + 갭 락
    2. STATEMENT 포맷의 바이너리 로그를 사용하는 MySQL 서버에서는, REPEATABLE READ 격리 수준을 사용해야 한다. 또한 innodb_locks_unsafe_for_binlog 시스템
       변수가 비활성화 되면 (0으로 설정되면) 변경을 위해 검색하는 레코드에는 넥스트 키 락 방식으로 잠금이 걸린다. InnoDB의 갭 락이나 넥스트 키 락은 바이너리 로그에 기록되는 쿼리가 레플리카 서버에서
       실행될 때 소스 서버에서 만들어 낸 결과와 동일한 결과를 만들어내도록 보장하는 것이 주목적이다. 그런데 이외로 넥스트 키 락과 갭 락으로 인해 데드락이 발생하거나 다른 트랜잭션을 기다리게 만드는 일이
       자주 발생한다. 가능하다면 바이너리 로그 포맷을 ROW 형태로 바꿔서 넥스트 키 락이나 갭 락을 줄이는 것이 좋다.
4. 자동증가 락
    1. 자동증가시, 동시에 여러 레코드가 INSERT 되어 중복이 되지 않도록 관리하는 락
    2. 새로운 레코드를 저장하는 쿼리에서만 필요 → update나 delete에서는 걸리지 않음
    3. 트랜잭션과 관계없음, AUTO_INCREMENT값을 가져오는 순간에 걸림
    4. 아주 짧게만 적용되기 때문에, 명시적 선언 X
    5. 5.1 이상부터는 , 모든 INSERT 문장에 자동증가 락을 사용하는 것이 아니라 INSERT 되는 레코드 수를 정확히 예측할 수 있을 때 사용하지 않음. 뮤텍스 이용해 처리
    6. 자동증가 값이 증가하면 절대 줄어들지 않는 이유가 AUTO_INCREMENT잠금을 최소화하기 위해서다. INSERT 쿼리가 실패했더라도 한번 증가된 AUTO_INCREMENT값은 다시 줄어들지 않고
       그대로 남는다.

인덱스와 장금

레코드를 잠그는것과, 레코드의 인덱스를 잠그는 것의 차이

https://www.youtube.com/watch?v=onBpJRDSZGA

## MySQL 의 격리수준

트랜잭션이란, Commit or Rollback
트랜잭션의 특징 ACID 중 I는 Isolation으로, 각 트랜잭션들이 서로에게 영향을 주지 않도록
-> 트랜잭션이 독립적으로 실행되는 것처럼 보이게 만든다.

Read Uncommitted -> 커밋되지 않은 데이터에 접근 가능
Read Committed -> 커밋된 것만 ㅇ

4개의 격리 수준에서, 순서대로 뒤로 갈수록 각 트랜잭션간의 데이터 격리정도가 높아지며, 동시 처리 성능도 떨어지는것이 일반적이라고 볼 수 있다. 격리수준이 높아질 수록 서버처리성능이 많이 떨어질 것으로 생각하는
사용자가 많은데, SERIALIZALBE 격리 수준이 아니라면 큰 차이가 없다.

|                  | Dirty Read | Non-Repeatable Read | Phantom Read    |
|------------------|------------|---------------------|-----------------|
| Read Uncommitted | 발생         | 발생                  | 발생              |
| Read Committed   | 없음         | 발생                  | 발생              |
| Repeatable Read  | 없음         | 없음                  | 발생 (InnoDB는 없음) |
| Serializable     | 없음         | 없음                  | 없음              |

Dirty Read : 커밋되지 않은 트랜잭션의 데이터를 읽는 현상 → 다른 트랜잭션이 데이터를 수정했지만, 아직 커밋되지 않은 상태에서 데이터를 읽을 때 발생 -> 롤백하기 전 데이터를 볼 수 있음

Non-Repeatable Read : 한 트랜잭션 내에서 같은값을 두 번 읽었을 때, 다른 트랜잭션이 해당 데이터 수정시 값이 다름

Phantom Read : 한 트랜잭션 내에서 같은 쿼리를 두 번 실행시, 다른 트랜잭션이 데이터를 삽입하여 두 번째 쿼리 결과가 다름
select * from user where id >=4 for update 라는 쿼리를 실행 시,
해당 id >=4 인 레코드들에 대해 잠금을 걸게 됨 -> 변경 불가능
하지만, 새로운 레코드의 삽입을 막지는 못해서,
id = 7인 새로운 레코드를 삽입한다면,
동일한 쿼리인 select * from user where id >=4 for update 라는 쿼리를 실행해도
같은 트랜잭션 내에서 결과값이 다를 수 있음

1. Read Uncommitted
    1. commit, rollback 상관없이 각 트랜잭션의 변경 내용이 다른 트랜잭션에서 보임
    2. 롤백해도, 롤백하기전 내용을 읽을 수 있음
2. Read Committed
    1. 가장 많이 사용
    2. MVCC
        1. 변경 수행시, 언두영역에 백업하여, Commit 이전이라면 SELECT 쿼리 결과가 언두 영역에 백업된 레코드에서 가져옴
        2. 언두영역이 있는데, 왜 non-repeatable read 부정합이 발생할까??
            1. 한쪽이 select 두 번 하는 사이, 다른쪽에서 update 및 commit 을 한 경우
            2. 변경후 커밋까지 한 경우!
            3. 커밋을 하지 않았다면, 언두영역에서 가져옴
3. Repeatable Read
    1. 트랜잭션 내 SELECT vs 그냥 SELECT
    2. REPEATABLE READ 단계에서는, 트랜잭션이 시작되고 끝나기 전 모든 SELECT가 동일한 결과 반환
    3. 동일 트랜잭션 내에서는 동일한 결과
    4. 자신의 트랜잭션 번호보다 작은 트랜잭션 번호의 값만 읽어오는것
    5. 몇 번 째 이전 버전까지 찾아들어가는가? 가 Read Committed와의 차이
    6. 트랜잭션 시작 ID가 10 이라면, 해당 트랜잭션 안에서 실행되는 모든 쿼리는 10보다 작은것만 보게됨
    7. 언두영역의 레코드에는 트랜잭션 번호가 포함되어 있음, 불필요할때 언두영역 삭제
    8. SELECT 하는 레코드에 쓰기잠금을 걸어야 하는데, 언두레코드에는 잠금을 걸 수 없어서
4. Serializable
    1. 읽기마저 LOCK을 거는것
    2. 레코드를 읽기만 해도 다른 트랜잭션에서 변경 못함

https://www.youtube.com/watch?v=QHWwNTGkwAU
