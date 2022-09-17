# 4.2 InnoDB 스토리지 엔진 아키텍처 (1)

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/f0dc9e2a-4682-44ae-8ce3-13ad4075e627/Untitled.png)

- MySQL 스토리지 엔진 중 유일하게 **레코드 기반의 잠금을 제공**

# 4.2.1 프라이머리 키에 의한 클러스터링

- 모든 테이블은 **프라이머리 키를 기준으로 클러스터링** 되어 저장됨
  - = 프라이머리 키 값의 순서대로 디스크에 저장
  - = 모든 **세컨더리 인덱스는** 레코드의 주소 대신, **프라이머리 키의 값을 논리적 주소**로 사용
- 프라이머리 키 = 클러스터링 인덱스
  - 프라이머리 키를 이용한 레인지 스캔이 빨리 처리됨
  - 그러므로, 쿼리 실행계획에서 프라이머리키는 다른 보조 인덱스에 비해 비중이 높게 설정됨!

  이는, 오라클의 IOT(Index organized table)과 동일한 구조임.


> MyISAM 은 클러스터링 키 지원 X
- 프라이머리 키와 세컨더리 인덱스의 구조적인 차이가 없음
- 프라이머리 키 = 유니크 제약을 가진 세컨더리 인덱스
- 모든 인덱스는 물리적인 레코드 주소값(ROWID)를 가짐
>

* `프라이머리 인덱스` VS `세컨더리 인덱스(PK이외의 인덱스`)

* `클러스터링 인덱스(순서O)` VS `Non-클러스터링 인덱스(순서X)`

* InnoDB에서는 `프라이머리 인덱스` = `클러스터링 인덱스` 라고 봐도 무방

# 4.2.2 외래 키 지원

- InnoDB에서만 사용가능 (MyISAM, MEMORY에선 사용불가)

운영의 불편함이 있지만, 개발환경DB에서는 좋은 가이드 역할

- 부모/자식 모두 외래키에 대한 인덱스 생성 필요
- 변경시, 부모/자식 테이블에 데이터 있는 체크 작업 → locking이 전파됨 → 데드락 발생 *주의
- 수동적으로 데이터 적재 + 스키마 변경 관리시 실패 가능성 발생 (외래키 관계를 식별해 순서대로 작업 필요)
  - `SET foreign_key_checks = OFF` : 외래키 관계 체크 일시적 중단.

    → DELETE/UPDATE CASCAD도 무시됨

  - `SET **SESSION** foreign_key_checks = OFF`
    - GROBAL/SESSION 옵션 줄수 있음. 가능한 SESSION으로 작업 권장.

# 4.2.3 MVCC(Muti Version Concurrency Control)

> 레코드 레벨의 트랜젝션을 지원
목적 : 잠금을 사용하지 않는 **일관된 읽기 제공**
>

- InnoDB는 Undo log를 이용해 해당 기능을 구현
- Muti Version = 하나의 레코드에 여러 버전이 동시에 관리됨

## 격리수준에 따른 읽기 - MySQL + InnoDB storage engine

1. INSERT

`INSERT INTO member (m_id, m_name, m_area) VALUES (12, ‘홍길동’, ‘서울’);`

`COMMIT;`

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/47b13af5-1626-48c3-9cb5-a5ad7f95df93/Untitled.png)

1. UPDATE

`UPDATE member SET m_area=’경기’ WHERE m_id=12;`

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/24d65f29-30b2-4d8a-ad9b-0aab4642f8e1/Untitled.png)

: 일반적으로 버퍼풀 과 디스크는 ACID보장에 따라 동일한 상태라고 가정할 수 있다.

### COMMIT or ROLLBACK전, 다른 사용자가 데이터를 읽는다면?

⇒ 격리 수준에 따라서 다르다!

- READ_UNCOMMITED : `버퍼 풀`이 가지고 있는 변경된 데이터를 반환
- READ_COMMITED (이상의 격리 수준, REPEATABLE_READ, SERIALIZABLE) : 아직 커밋 전이므로, 이전 내용을 보관하는 `언두 로그`의 데이터를 반환

> **언두 로그**
: 트랜잭션이 길어지면, 언두 로그의 데이터가 삭제되지 못하고, 공간이 많이 늘어날 수 있다.
- COMMIT : 언두 영역의 데이터 → 디스크에 저장
- ROLLBACK : 언두 영역의 데이터 → 버퍼 풀로 복수 → 언두 영역 데이터 삭제
  **데이터 삭제 시점 ?** 해당 데이터를 필요로하는 트랜잭션이 더 이상 없을때 비로소 삭제가 된다.
>

# 4.2.4 잠금 없는 일관된 읽기(Non-Locking Consistent Read)

- InnoDB는 MVCC기술을 통해 잠금 없이 읽기 수행 가능

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/4e79e41c-3f41-4d7f-b7f8-c7fd455dbb6d/Untitled.png)

- SERIALIZABLE
- READ_UNCOMMITED, READ_COMMITED, REPEATABLE_READ

  → INSERT와 연결되지 않은 읽기(SELECT) 다른 트랜잭션의 변경과 관계없이, 항상 잠금 대기 하지 않고 실행


> 일관된 읽기를 위해, 트랜잭션이 활성 상태일때 Undo log를 지우지 못함
트랜잭션이 길어진다면, 해당 데이터를 유지하는것에 따른 문제가 발생할 수 있으므로, 커밋이나 롤백을 통해 가능한 빠르게 트랜잭션을 종료 할 수 있도록 하는것이 좋다.
>

# 4.2.5 자동 데드락 감지

> InnoDB는 내부적으로 데드락을 방지하기 위한 `Wait-for List (잠금대기목록)`을 **그래프 형태**로 관리
>
> - `데드락 감지 Thread`가 주기적으로 **잠금 대기 그래프를 검사**
> - 교착상태에 빠진 트랜잭션들을 찾아 그 중 하나를 **강제 종료**

- 강제종료 판단 기준?
  - **언두로그 레코드를 더 적게** 가진 트랜잭션이 일반적인 롤백의 대상이됨
  - 언두로그 가 적다 = 롤백 처리양이 적다 = 서버 부하가 적다

- `Innodb_table_locks` : 테이블 잠금 접근 허용
  - 상위 레이어인 MySQL 엔진에서 관리되는 **테이블 잠금(LOCK TABLE 명령으로 잠긴것)** 접근 불가
  - 이로 인해 데드락 감지 정확성 문제 발생 가능성 존재
  - 가능한, **Innodb_table_locks 시스템 변수 활성화**를 통해 이를 방지 하는것 권장

- 데드락 감지 스레드
  - 잠금목록 검사를 위해, **잠금 테이블(잠금 목록 저장 리스트)** 에 잠금을 건다.
  - **데드락 감지 스레드가 느려진다면?**

    (대게는 빠르지만, 동시 처리 스레드가 많거나, 트랜잭션이 가지는 잠금의 수가 많아지면 느려질 수 있다.)

  - 서비스 처리 중이던 스레드가 작업을 하지 못하고 대기 → 서비스에 악영향

**PK or 세컨더리 인덱스를 기반으로 매우 높은 동시성 처리를 요구하는 서비스가 있다면,**

- `innodb_deadlock_detect` : 데드락 감지 스레드 동작 제어
  - 해당 옵션을 OFF로 성능 비교 권장
- `innodb_lock_wait_timeout` : 일정시간 이후, 자동으로 요청 실패 후 에러 메시지 반환
  - 초단위로 설정 가능
  - **innodb_deadlock_detect OFF시** 무한대기에 빠지는것을 방지 하기 위해, 50초 보다 낮게 설정 권장

# 4.2.6 자동화된 장애 복구

> InnoDB는 MySQL서버 시작시에, 완료되지 못한 트랜잭션, 디스크에 일부만 기록된 데이터 페이지에 대한 복구 작업을 자동으로 진행
>

대게 이슈가 없지만, MySQL서버와 무관하게 디스크나 서버 하드웨어 이슈로 인해 **자동복구 실패시**

→ MySQL서버가 시작되지 않음

1) **MySQL서버 시작**

`innodb_force_recovery` : InnoDB 스토리지 엔진이 데이터 파일이나 로그 파일의 손상 여부 검사 과정을 선별적으로 진행

- 이외, 1~6 까지 변경하며 MySQL서버 기동 (숫자가 클 수록 손상도가 큰것임)
- 0이 아닌, 복구 모드(1~6)에서는 SELECT 이외 INSERT, UPDATE, DELETE 수행 부
- **옵션에 따른 장애상황 및 해결방법**

  ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/3642a2fb-b13f-4ef6-89a0-f87038b0ffd0/Untitled.png)

  ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/56ea8db0-c037-43f5-9e88-317ca8f29334/Untitled.png)


1~6까지 해봤는데도 안되면, 백업후 다시 구축하는 방법뿐이다.

마지막 풀 백업 + 바이너리 로그 를 통해 최대한 장애시점 까지의 데이터를 복구

2) **백업**

`mysqldump`

서버 기동후 데이터를 가능한 만큼 백업 후 → MySQL서버의 DB와 테이블을 다시 생성하는 것 권장

# 4.2.7 InnoDB 버퍼 풀

(InnoDB스토리지 엔진의 가장 핵심)

> **디스크의 데이터 파일** or **인덱스 정보**를 메모리에 캐시해 두는 공간
- 쓰기지연 버퍼 역할 ⇒ 디스크 작업 횟수 감소
>

### 1. 버퍼 풀의 크기 설정

`innodb_buffer_pool_size`

- 일반적으로 물리 메모리의 80% ? Nope OS와 각 클라이언트 스레드가 사용할 메모리를 충분히 고려해서 설정해야함 (50%정도 주고 80%까지 늘려가며 정적선 찾아야함)
- 대게 MySQL서버내 메모리 필요로 하는 부분이 없지만, 종종 레코드 버퍼가 상당한 메모리 사용
  - 레코드 버퍼? 각 클라이언트 세션에서 테이블의 레코드를 읽고 쓸 때 버퍼로 사용하는 공간
  - 커넥션이 많고, 사용 테이블이 만다면 메모리가 많이 필요함
  - 이는 별도설정 불가
  - 전체 커넥션 개수 + 각 커넥션에서 읽고 쓴  테이블 개수에 따라서 결정됨. 또한 이는 동적으로 해제 되서 정확한 공간 크기 계산이 어려움

  **MySQL 5.7부터는 동적으로 버퍼 풀 크기를 조절 가능**

  - 작은 값으로 설정후 상황에 따라 증가시킴
  - 크리티컬한 변경임. 한가한 시점에 진행.
  - 크게하는거 보단 줄이는게 영향도가 더 높음. (가능한 줄이는건 지양)
  - 내부적으로 128MB청크 단위로 쪼개서 관리됨. (늘리거나 줄일때 단위)

`innodb_buffer_pool_instances` : 버퍼 풀을 쪼개서 관리

- 원래 8개 정도로 초기화 되나, 버퍼 풀을 위한 메모리가 1GB미만이면 1개만 생성됨

### 2. 버퍼 풀의 구조

### 3.버퍼 풀과 리두 로그

### 4. 버퍼 풀 플러시(Buffer Pool Flush)

### 1) 플러시 리스트 플러시

### 2) LRU 리스트 플러시

### 5. 버퍼 풀 상태 백업 및 복구

### 6. 버퍼 풀의 적재 내용 확인
