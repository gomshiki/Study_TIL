# [CS스터디] 데이터베이스

# 1. Trigger

## 1) SQL 의미

- 트리거의 의미 : 방아쇠, 촉발시킴
- SQL에서 의미 : 데이터베이스에서 어떤 이벤트가 발생했을 때 자동적으로 실행되는 프로시저(procedure)
    - DB에서 insert, update, delete 수행 시 자동적으로 실행되는 프로시저를 의미

### Trigger 예시1

```sql
// Mysql 
delimiter $$

// Trigger 구현
CREATE TRIGGER log_user_nickname_trigger // trigger 명명

BEFORE UPDATE // update 이전에 수행
ON users FOR EACH ROW // on <타켓테이블>, FOR EACH ROW : update되는 각 row에 대해 trigger tngdo
BEGIN
	insert into user_log values(OLD.id, OLD.nickname, now()); // 수행할 쿼리(Action)
END
$$

delimiter;

```


> OLD.id, OLD.nickname : update 되기 전 tuple을 가리킴
> 
> BEFORE DELETE 인 경우 : delete된 tuple을 가리킴

<br>

### Trigger 예시2

- user_id 1 인 사용자가 구매시 총 구매 합계를 테이블에 저장하도록 트리거 시킴

```sql
delimiter $$
CREATE TRIGGER sum_buy_prices_trigger // trigger 명
AFTER INSERT // INSERT 이벤트가 발생한 후에
ON buy FOR EACH ROW // buy 테이블에서 각 행마다
BEGIN
	DECLARE total INT; // INT 타입의 total 변수 선언
	
	// INT 타입의 user_id 선언 및 buy 테이블 user.id 값으로 초기화
	DECLARE user_id INT DEFAULT NEW.user_id; 
	
	// buy 테이블의 총합(sum)을 total 변수에 저장
	select sum(price) into total from buy where user_id = user_id;
	
	// user_buy_stats 테이블의 prices_sum에 앞서 저장한 total 값으로 update
	update user_buy_stats set prices_sum = total where user_id = user_id;
	
END
$$
delimiter;
```


> NEW 키워드 : insert된 tuple을 가리킴, update된 후의 tuple을 가리킴


<br>

### Trigger 정의 시 알고 있으면 좋은 내용1

> update, insert, delete 등을 한번에 감지하도록 설정가능 ( MySQL 불가능)

- PostgreSQL 예시

```sql
# postgresql
CREATE TRIGGER avg_empl_salary_trigger
	AFTER INSERT OR UPDATE OR DELETE // INSERT, UPDATE, DELETE 중 한개라도 수행 시
	ON employee
	FOR EACH ROW
	EXECUTE FUNCTION update_avg_empl_salary();
```

> `FOR EACH ROW` 문제점
> 
> employee 테이블에 update가 5번 수행된다면 Trigger 도 5번 수행 ⇒ 비효율적!!
> 
> `FOR EACH STATEMENT를` 사용해 update 1번에 Trigger 도 1번 수행하도록 변경


<br>

### Trigger 정의 시 알고 있으면 좋은 내용2

> row 단위가 아닌 statement 단위로 trigger를 수행하도록 설정 가능
> 
> (MySQL은 FOR EACH STATEMENT 사용 불가능)


```sql
# postgresql
CREATE TRIGGER avg_empl_salary_trigger
	AFTER INSERT OR UPDATE OR DELETE // INSERT, UPDATE, DELETE 중 한개라도 수행 시
	ON employee
	FOR EACH STATEMENT
	EXECUTE FUNCTION update_avg_empl_salary();
```

<br>

### Trigger 정의 시 알고 있으면 좋은 내용3

> 💡 trigger 발생시킬 조건을 세부적으로 지정 가능 (MySQL 불가능)

```sql
#postgresql
CREATE TRIGGER log_user_nickname_trigger
	BEFORE UPDATE
	ON users
	FOR EACH ROW
	
	// 기존 닉네임과 변경 닉네임이 동일하면 트리거 수행 X
	WHEN (NEW.nickname IS DISTINCT FROM OLD.nickname)
	EXECUTE FUNCTION log_user_nickname();
```

<br>

### trigger 사용 시 주의사항

> 트리거는 `소스코드로 발경할 수 없는 로직`이기 때문에 
> 어떤 동작이 일어나는지 파익이 어렵고, 이슈 발생시 대응이 어려움.
- 과도한 트리거 사용은 DB 성능 저하를 야기
- 디버깅 어려움
- 문서 정리가 매우 중요!

### 결론 : 트리거는 최후의 카드로 사용 권장 => 사용하지 마셈

---

# 2. Schedule과 Serializability

## Schedule

- 여러 트랜잭션들이 동시에 실행될 때 각 트랜잭션에 속한 오퍼레이션(Operation)들의 실행순서
- 각 트랜잭션 내의 오퍼레이션들의 순서는 바뀌지 않음
- Operation : read, write 와 같은 일련의 작업

- sched.4 : r1(K) w1(K) r1(H) r2(H) w2(H) c2 w1(H) c1 의미
    - r1(K) : read작업 + 1번 트랜잭션 + (K의 balance : 잔액)
    - w1(K) : write작업 + 1번 트랜잭션 + (K의 balance : 잔액)
    - r2(H) : read작업 + 2번 트랜잭션 + (H의 balance : 잔액)
    - w2(h) : write작업 +  2번 트랜잭션 + (H의 balance : 잔액)
    - c1, c2 : commit 작업 + 1번 트랜잭션, commit 작업 + 2번 트랜잭션

### Serial Schedule

- Serial schedule 성능
    - `한번에 하나의 트랜잭션만 수행`해 좋은 성능은 낼수 없고, 현실적을 사용할 수 없는 방식


### Nonserial Schedule

- Nonserial Schedule 성능
    - 트랜잭션들이 겹쳐서 실행되기에 `동시성이 높아지고 같은 시간 동안 더많은 트랜잭션을 처리`할 수 있음
- 단점
    - 트랜잭션들이 겹쳐 수행되는 형태에 따라 이상한 결과가 발생할 수 있음


### Conflict : 두 개의 오퍼레이션에 대해 사용하는 개념

- 아래 조건을 만족하면 conflict
    - 서로 다른 트랜잭션 소속
    - 같은 데이터에 접근
    - 최소 하나는 write operation 수행
- conflict 오퍼레이션은 순서가 바뀌면 결과도 바뀜



### Conflict equivalent

- 아래 두조건을 모두 만족
    - 두 schedule은 같은 트랜잭션을 가진다
    - 두 schedule의 operation 순서(서로다른 트랜잭션 + 같은 데이터  + 최소1개는 write)는 동일해야한다

### Conflict serializable

- serial schedule과 conflict equivalent일 때

### Nonserial Schedule을 통한 RDBMS 성능 향상 방법



### 정리

- 어떤 schedule이 어떤 임의의 serial schedule과 동일하다면(`equivalent`)하다면 
  - 해당 schdule은 `serializability` 속성을 가진다.
- 어떤 schedule이 어떤 임의의 serial schedule과 `conflict equivalent` 하다면 
  - 해당 schdule은 `conflict serializability` 속성을 가진다.
- 어떤 schedule이 어떤 임의의 serial schedule과 `view equivalent` 하다면 
  - 해당 schdule은 `view serializability` 속성을 가진다.

---

# 3. recoverable schedule, cascadeless schedule, strict schedule

## 1) unrecoverable schedule

- schedule 내 commit 된 트랜잭션이 rollback 된 트랜잭션이 write 했었던 데이터를 읽은 경우
- rollback을 해도 이전 상태로 회복 불가능 할 수 있기에 DBMS에서 허용하면 안됨


## 2) recoverable schedule

- schedule 내에 어떤 트랜잭션도 자신이 읽은 데이터를 write 한 트랜잭션이 먼저 commit/rollback 하기 전까지 commit 하지 않는 경우
- rollback 할때 이전 상태로 온전히 돌아갈 수 있기에 DBMS 는 이런 schedule 만 허용해야함



## 3) cascading rollback

- 하나의 트랜잭션이 rollback 하면 의존성이 있는 다른 트랜잭션도 rollback 해야함
- 여러 트랜잭션의 Rollback이 연쇄적으로 일어나면 처리비용이 많이 듬.


## 4) cascadeless schedule (= avoid cascading rollback)

- schedule 내에서 어떤 트랜잭션도 commit 되지 않은 트랜잭션들이 write 한 데이터를 읽지 않는 경우


- 아래와 같이 read를 수행하지 않아도 cascadeless schedule이라고 할수 있음.



## 5) Strict schedule

- schedule 내에서 어떤 트랜잭션도 commit 되지 않은 트랜잭션들이 write 한 데이터를 읽지 않는 경우
+ 쓰지도 않는 경우
- rollback 할 때 recovery가 쉬움. 트랜잭션 이전 상태로 돌려놓기만 하면됨



## 6) 정리

> - `recoverable schedule` : 어떤 트랜잭션이 write 한 데이터를 다른 트랜잭션이 read 했다면 앞선 write한 트랜잭션이 commit 할 때 까지는 read한 트랜잭션도 commit 하지 않을 때
rollback 시 회복 가능한 schedule
> - `cascadelesss schedule` : cascading rollback 을 발생하지 않게하는 schedule
> - `strict schedule` : commit 되지 않는 트랜잭션들이 write한 데이터는 read/write 하지 않는 schedule
> - `Isolation` : concurrency control(병행제어) 이 serializability와 recoverability를 제공할 때

---

# 4. 트랜잭션들이 동시에 실행될 때 발생가능한 이상현상들과 isolation level

### 1) Dirty read
- commit 되지 않은 변화를 읽음


### 2) Non-repeatable read(=Fuzzy read)
- 같은 데이터의 값이 달라짐


### 3) Phantom read
- 없던 데이터가 생김



### 4) Isolation level

> 💡 Dirty read, Non-repeatable read, Phantom read와 같은 이상 현상이 모두 발생하지 않게 만들수는 있지만 제약사항이 많아져 동시 처리가능 트랜잭션수가 줄어듬 ⇒ DB 전체 처리량이 감소
> 
> **isolation level 등장 배경**
> 
> `일부 이상현상을 허용하는 몇 가지 level을 만들어 개발자가 필요에 따라 선택할 수 있도록 하자`

<br>

### Isolation level 정리 테이블
- Serializable : 아예 이상현상 자체가 발생하지 않는 level을 의미 ⇒ DB 전체 처리량 감소
> 💡 개발자는 isolation level을 통해 전체 처리량(throughput)과 데이터 일관성 사이에서 어느정도 거래(trade)를 할 수 있다.

- 하지만 위에서 정의한 3가지 외에 이상현상이 더 존재

### 5) Dirty write
- commit 안된 데이터를 write 함
- Rollback 시 정상적인 recovery는 매우 중요하기 때문에 모든 isolation level에서 dirty write를 허용하면 안된다.


### 6) Lost update
- update를 덮어씀

### 7) Dirty read  확장
- commit 되지 않은 변화를 읽음 + **abort가 발생하지 않아도 dirty read가 될 수 있다.**

### 8) Read skew
- inconsistent한 데이터 읽기

### 9) write skew
- inconsistent한 데이터 쓰기
- 예시 : x = 50, y = 50 제약조건 x+y≥0 / 트랜잭션1 : x에서 80을 인출 / 트랜잭션2 : y 에서 90을 인출

### 10) Phantom read 확장
- 예시 : 튜플1의 값중 v=7, v값 중 10보다 크면 cnt
트랜잭션1 : v > 10 데이터와 cnt를 읽는다 / 트랜잭션2 : v = 15인 t2를 추가하고 cnt를 1증가한다.


## Snapshot Isolation
> - type of MVCC : MultiVersion Concurrency Control
> - 트랜잭션 시작 전에 commit된 데이터만 보임
> - 처음 커밋한 트랜잭션이 winner ⇒ 동일한 데이터를 동시에 쓰기작업시 늦게 커밋한 트랜잭션은 abort
> - read한 데이터를 snapshot으로 만들어 다른 트랜잭션에 의해 데이터가 변경되더라도 변경되기 전 데이터를 읽도록 함.

## RDBMS 별로 제공되는 isolation level

> 💡 주요 RDBMS은 SQL 표준에 기반해서 isolation level을 정의
> 
> RDBMS마다 제공하는 isolation level 다름
> 
> 같은 이름의 isolation level이라도 동작 방식이 다름
> 
> `따라서, 개발자는 사용하는 RDBMS의 isolation level을 파악 후 적절한 level로 사용해야함`


- MySQL(innoDB) : 표준 SQL을 바탕으로 정의
- Oracle : 주 사용 level ⇒ READ COMMITED & SERIALIZABLE(READ UNCOMMITED 제공X)
    - Oracle에서 SERIALIZABLE Level은 SNAPSHOT Isolation 으로 동작
- SQL Server
    - 표준 SQL Isolation을 바탕으로 서비스
- PostgreSQL
    - 이상현상 : 표준 SQL 기반 3개 + Serialization Anomaly
    - Repeatable read = Snapshot Isolation

---

# 5. write-lock , read-lock, 2PL(two-phase locking) protocol

- write operation 은 단순히 값 하나 바꾸는 것보다 복잡한 과정을 거침
- 같은 데이터에 또다른 read/write 가 있다면 예상치 못한 동작이 발생할 수 있음
- 이를 해결할 수 있는 방법이 **`LOCK`**


## LOCK

데이터를 변경하거나 읽으려면 LOCK을 취득해야함.

### LOCK 예시1(Write Lock - Write Lock)

- 초기데이터 x : 10
- transaction1 이 먼저 write LOCK을 획득 / Transaction2 는 write LOCK 획득을 위해 대기 = BLOCK
- transaction1 이 LOCK을 해제 / Transaction2 는 write LOCK을 획득 후 write 수행
- Transaction2는 다시 write LOCK을 해제 ( 다음 트랜잭션 수행을 위해)

### LOCK 예시2(Write Lock - Read Lock)

- 초기데이터 : x = 10
- T1이 write lock을 취득 / T2는 read lock 획득을 위해 대기 = Block
- T1이 write 작업 후 Lock 해제 / T2가 read lock 획득


### 1) write-lock(exclusive lock)

- read/write(insert, update, delete) 할 때 사용
- 다른 tx가 같은 데이터를 read / write 하는 것을 허용하지 않음

### 2) read-lock(shared lock)
- read 할 때 사용
- 다른 tx가 같은 데이터를 read 하는 것만 허용(write 허용 x)

### read-lock 과 write-lock 예시1
- 초기 데이터 : x = 10
- T2 : read-lock 획득 / T1 : write-lock 요청했으나 대기
- T2 : read 완료 후 lock 해제 / T1 : write-lock 획득
- T1 : write 작업 후 write-lock 해제



### read-lock 과 read-lock 예시2
- 초기 데이터 : x = 10
- T2 : read-lock 획득 / T1 : read-lock 획득
- T2, T1 읽기 작업 후 read-lock 해제


---

## Lock 사용시에도 발생하는 이상현상 예제


> 💡 Lock을 사용했음에도 Serializability 를 보장하지 못하는 경우도 있음
> 
> Lock 사용 시에 발생할 수 있는 이상현상에 대해 알아보자



- 초기 값 : x= 100, y=200
- Serial schedule로 동작 : `t1 → t2 와  t2 → t1 결과 데이터가 다름`
- 초기 값 : x= 100, y=200
- Non-serial schedule로 동작
- 문제가 되는 부분 : T2에서 y 를 업데이트 하기 전에 T1에서 y 를 읽어서 문제가 발생
- 해결방법 : unlock(x)와 write_lock(y)의 순서를 바꿈 (write_lock 먼저 수행)
- 위와 같이 순서를 바꾸면 전체 트랜잭션 schedule 이 변경됨



## 2PL(two-phase locking) protocol

- tx에서 모든 locking operation이 최초의 unlock operation 보다 먼저 수행되도록 하는 것
    - Expanding phase(growing phase) : lock을 취득하기만 하고 반환하지 않는 phase
    - Shrinking phase(contracting phase) : lock을 반환만 하고 취득하지 않는 phase
- 순서 : write_lock(x) → unlock(y) / write_lock(y) → unlock(x)
- 2PL 프로토콜은 `Serializability를 보장`

### Deadlock
- 2PL 프로토콜은 Deadlock 이 발생할 수 있음

## 정리
- 트랜잭션의 isolation을 보장하기 위해 concurrency control(병행 제어) 이 존재
- Lock을 통한 Concurrency Control(병행제어) 구현방법
- 2PL protocol 를 기반으로 Lock을 설계해 RDBMS를 구현하면 Concurrency Control(병행제어)은  serializability를 보장함

---

### Conservative 2PL
- 모든 lock을 취득한 뒤 transaction을 시작
- Deadlock-free : Deadlock이 발생하지 않음
- 실용적이지 않음



### Strict 2PL(S2PL)
- strict schedule을 보장하는 2PL
- recoverability 보장
- write-lock을 commit / rollback 될 때 반환



### Strong Strict 2PL(SS2PL or rigorous 2PL)
- strict schedule을 보장하는 2PL
- recoverability 보장
- S2PL 보다 구현이 쉬움

---

# 6. MVCC와 isolation level과 함께 MVCC case study
(MySQL, postgreSQL)

- read - write 허용
- MVCC 특징1 : commite 된 데이터만 읽는다

### Isolation level :  Read commited
- read하는 시간을 기준으로 그 전에 commit된 데이터를 읽음


### Isolation level : Repeatable read
- 트랜잭션 시작 기준으로 그전에 commite된 데이터를 읽음
- (RDBSM 마다 조금씩 다름)

> 💡 MVCC 특징
> - 데이터를 읽을 때 특정 시점 기준으로 가장 최근에 commit된 데이터를 읽는다
> 
>   ⇒ `MySQL 에서는 이를 Consistent read라고 정의`
> - 데이터 변화(write) 이력을 관리한다.
> - read와 write는 서로를 block하지 않는다.

### Isolation level : Serializable
- MySQL : MVCC로 동작하지 않고 Lock으로 동작
- PostgreSQL : SSL(Serializable Snapshot Isolation) 기법으로 적용된 MVCC로 동작

### Isolation level : read uncommited
- MVCC는 commited된 데이터를 읽기 때문에 이 level에서는 보통 MVCC가 적용되지 않음


## PostgreSQL 에서 Lost Update 해결
> 💡 postgreSQL의 MVCC는 lost update를 repeatable read로 수행 시 해결 가능

> postgreSQL(Repeatable read) : 같은 데이터에 먼저 update 한 트랜잭션이 commite 되면
나중 트랜잭션은 Rollback 된다.



> 💡 Lost Update 를 해결하기 위해서는 한 트랜잭션 isolation level 뿐만아니라 연관된 다른 트랜잭션 isolation level도 함께 고려해야함.

## MySQL 에서 Lost Update 해결

- Locking read : 가장 최근의 commit 된 데이터를 읽음
    - isolation level 과 상관없이 수행 (repeatable read 무시)
    - repeatable read : 트랜잭션이 시작하는 순간을 기준으로 데이터를 읽음( x = 50)

> 💡 MySQL에서 Lost Update를 방지하기 위해서는 Locking read를 사용해야함.
> 
> repeatable read(Isolation level)로는 해결되지 않음
> 
> Serializable level이면 해결가능.

<br>

### Locking read 적용 예시
```sql
SELECT ... FOR UPDATE; // write lock(exclusive lock) 획득
SELECt ... FOR SHARE; // read lock(shared lock) 획득
```


### MySQL 
- Locking read 사용

### PostgreSQL

- repeatable read : 같은 데이터에 먼저 update한 트랜잭션이 commit 되면 나중 트랜잭션이 rollback 됨
- PostgreSQL도 Mysql과 동일한 문법 존재

```sql
SELECT ... FOR UPDATE;
SELECT ... FOR SHARE;
```


### Serializable level 에서 Mysql 과 PostgreSQL 동작 비교

### MySQL
> 💡 repeatable read 와 유사
> 
> 트랜잭션의 select 문은 암묵적으로 select … for share 처럼 동작(read lock)
- 
- MySQL에서 serializable level은 MVCC 보다 LOCK으로 동작
- for update가 아닌 for share인 이유 : 성능때문
    - 장점 : for update(exclusive lock) 사용 시 성능 이슈가 있을 수 있음
    - 단점 : deadlock 발생 가능성 높음

### PostgreSQL

> 💡 SSI(Serializable Snapshot Isolation)로 구현 : MVCC로 동작하면서 모든 이상현상을 막아줌
> 
> `first-committer-winner`

---

### 참고 강의 - 쉬운코드의 데이터베이스 강의
[Youtube - 쉬운코드](https://www.youtube.com/watch?v=mEeGf4ZWQKI&list=PLcXyemr8ZeoREWGhhZi5FZs6cvymjIBVe&index=13)

