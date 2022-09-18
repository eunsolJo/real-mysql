# 4.2 InnoDB 스토리지 엔진 아키텍처 (2)

# 4.2.8 Double Write Buffer

리두로그

- 공간의 낭비를 막기 위해 **페이지의 변경된 내용만 기록**
- 이로인해, 더티 페이지 에서 디스크 파일로 플러시때, 일부만 기록되는 문제 발생시 **페이지 복구가 어려움**

  ### → 파셜 페이지(Partial-page) or 톤 페이지(Torn-page)

  - 페이지가 일부만 기록되는 현상
  - 하드웨어의 오작동 or 시스템의 비정상 종료로 인해 발생

### 위 문제를 막기 위한, Double-Write 기법

1. A~E 까지의 더티 페이지를 디스크로 플러시
2. 실제 데이터 파일에 변경내용 기록전, A~E 더티 페이지를 묶어 → 한번의 디스크 쓰기로 시스템 테이블스페이스의 DoubleWrite 버퍼에 기록
3. 이후, 각 더티 페이지의 파일을 적당한 위치에 랜덤으로 쓴다.

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/6b6e772b-8f46-40ff-89cf-87370fc6dfec/Untitled.png)

- DoubleWrite 버퍼 공간에 기록된 내용은 더티 페이지가 정상적으로 기록되면 필요없어짐.
- 중간에 실패했을때만 사용됨.

  → 가정) A,B는 정상 기록. C기록 도중 OS비정상 종료됨

  1. InnoDB 스토리지 엔진은 재시작시 항상,
  2. DoubleWrite 버퍼내용과 데이터 파일의 페이지들을 모두 비교
  3. 다른 내용을 담은 페이지 존재시, DoubleWrite → 데이터 파일 페이지로 복사
- `innodb_doublewrite` 시스템 변수로 사용여부 제어

### DoubleWrite at `HDD VS SSD`

- HDD
  - 자기 원판(Platter)이 회전 하는 저장 시스템
  - 한 번의 순차 디스크 쓰기 → 적합
- SSD
  - 랜덤 IO 나 순차 IO의 비용이 비슷 → 부적합

⇒ 그러나 결과적으로 데이터의 무결성이 중요하다면, DoubleWrite 활성화를 권장

DB Server성능을 위해 InnoDB 리두 로그 동기화 설정(innodb_flush_log_at_trx_commit = 1 : 활성)을 비활성화 하였다면 DoubleWrite 도 비활성 권장

# 4.2.9 언두 로그

: 트랜잭션 및 격리 수준 보장을 위해 DML(INSERT, UPDATE, DELETE) **변경전 데이터**를 별도로 백업한 데이터

- **트랜잭션** 롤백 대비용
- 트랜잭션 **격리 수준** 유지 및  높은 동시성 제공

언두로그는 중요하며, 비용이 많이 든다.

- 트랜잭션을 커밋하지 않아도, 실제 데이터 파일(데이터/인덱스 버퍼) 내용은 변경됨

### 4.2.9.1 언두 로그 레코드 모니터링

MySQL 5.5 이전 버전, 한번 증가한 언두 로그 공간은 다시 줄어 들지 않음

- ex. 100GB 데이터를 DELETE → 100GB의 언두로그 공간 생김
- 서버를 새로 구축하지 않으면, 줄어들지 않음
- 디스크 사용량 뿐만 아닌, 백업시에 해당 공간을 복사하기 위한 리소스 발생

### 대용량 or 오래걸리는 트랜잭션

- 언두로그의 양은 급격히 증가하며 바로 삭제가 어려움.
- 오래걸리는 트랜잭션
  - A——————————————————————→
  - —-B ——→
  - ———C ——→
  - B,C가 끝나도, A가 종료 되지 않아 B,C의 언두로그 또한 삭제할 수 없다.
  - 만약, 이게 하루동안 지속 된다면? 조회 쿼리를 날릴때, 언두 로그 이력을 모두 읽어야하므로 쿼리 성능이 저하됨

MySQL 5.7 - 8.0 버전,

- 언두로그를 돌아가면서 순차적으로 사용
- 필요한 시점에 자동으로 공간을 줄여줌
- 그럼에도 **장시간 트랜잭션**은 성능상 이슈 발생할 수 있음

  ### → 언두로그의 레코드수를 모니터링 하는것으로 방지

    ```sql
    -- for all version
    SHOW ENGINE INNODB STATUS \G;
    
    -- after 8.0
    SELECT count
      FROM information_schema.innodb_metrics
     WHERE SUBSYSTEM='transaction' AND NAME = 'trx_rseg_history_len';
    ```

  - DML건수에 따라 다름
  - 서버별 안정적 시점의 레코드 건수를 파악해 이를 기준으로 모니터링

### 4.2.9.2 언두 테이블스페이스 관리

MySQL 5.6이전 버전

- 시스템 테이블스페이스(ibdata.idb)에 저장
- 이는, MySQL 서버 초기화시 생성됨 → 확장에 한계

MySQL 5.6버전

- innodb_undo_tablespaces 시스템 변수 도입
- 2보다 크게 설정시, 별도의 언두 로그 파일을 사용
- 0설정시, 기존 처럼 동작

MySQL 8.0 (8.0.14~)

- ~~innodb_undo_tablespaces 효력 없어짐~~
- 항상 외부 별도 로그파일에 기록
- 언두 테이블스페이스를 동적으로 추가 삭제 가능

    ```sql
    -- 추가/삭제
    CREATE UNDO TABLESPACE extra_undo_003 ADD DATAFILE '/date/undo_dir/undo_003';
    DROP UNDO TABLESPACE extra_undo_003;
    
    -- 비활성화
    ALTER UNDO TABLESPACE extra_undo_003 SET INACTIVE;
    
    -- 파일확인
    SELECT TABLESPACE_NAME, FILE_NAME
      FROM INFORMATION_SCHMA.FILES
     WHERE FILE_TYPE LIKE 'UNDO LOG';
    ```


### 언두 테이블스페이스(Undo Tablespace)

: 언두 로그가 저장되는 공간

![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/ceba0210-8fab-47c6-9c5a-f79efbde2f54/Untitled.png)

- 하나의 언두 테이블스페이스는 1~128개의 롤백 세그먼트를 가짐
- 롤백 세그먼트는 1개 이상의 언두 슬롯을 가짐
  - (InnoDB 페이지 크기를 16byte로 나눈 값의 개수) 의 언두 슬록을 가짐
  - ex.
    - InnoDB 페이지 크기 = 16KB
    - 롤백 세그먼트는 1024개의 언두 슬롯을 가짐
- 하나의 트랜잭션에 필요로 하는 언두 슬록의 개수는 최대 4개이다.
  - 일반적으로 트랜잭션이 임시 테이블을 사용하지 않으므로, 대략 2개
  - (최대 동시 트랜잭션수) = (InnoDB 페이지 크기)/16*(롤백 세그먼트 개수)*(언두 테이블스페이스 개수)
    - 일반적인 옵션시, 131072(=16*1024/16*128*2/2) 개의 트랜잭션이 동시에 처리가능
    - 디폴트를 사용하는것 권장. 언두로그 남는게 낫지 부족하면 트랜잭션 시작 못함

### Undo tablespace truncate

: 불필요하게 할당된 언두 테이블스페이스 공간을 필요한 만큼만 남기고 OS로 반납

- 자동모드
  - 퍼지 스레드(Purge Thread)가 주기적으로 언두 퍼지(Undo Purge - 불필요한 언두로그 공간을 OS로 반납)를 한다.
  - innodb_undo_log_tuncate = ON : 사용여부
  - innodb_purge_rseg_truncate_frequency : 발생 빈도
- 수동모드
  - innodb_undo_log_tuncate = OFF : 사용여부
  - 자동모드가 효율적이지 않다고 판단되면, 언두 테이블스페이스를 비활성화 시킴 → 퍼지 스레드가 비활성화된 언두 테이블스페이스를 모두 OS에 반환
  - 단, 최소 3개 이상의 언두 테이블스페링스를 활성화 해두어야 한다.

    ```sql
    -- 언두 테이블스페이스 비활성화
    ALTER UNDO TABLESPACE tablespace_name SET INACTIVE; // 활성화 - ACTIVE
    ```


# 4.2.10 체인지 버퍼

### 체인지 버퍼(Change Buffer)

INSERT or UPDATE 시 데이터 파일 변경뿐만 아닌, **인덱스를 업데이트**해야함

- 인덱스 업데이트는 랜덤하게 디스크를 읽는 작업으로, 상당히 많은 자원을 소모함
- 변경해야할 인덱스 페이지가 버퍼 풀에 있으면? → 바로 업데이트
- 아니면? 임시공간(체인지 버퍼)에 저장한뒤, 사용자에게 결과를 바로 반환

### **유니크 인덱스는 체인지 버퍼를 사용 할 수 없다.**

사용자에게 결과를 반환전, 반드시 중복 여부 체크가 필요하므로

### 체인지 버퍼 머지 스레드(Merge Thread)

체인지 버퍼에 임시로 저장된 인덱스 레코드 조각은,

이후 백그라운드 스레드(체인지 버퍼 머지 스레드)에 의해 병함됨

MySQL 5.5 이전,

- INSERT만 가능
- 기본적으로 활성화 (시스템변수 없음)

MySQL 8.0 부터,

- INSERT, DELETE, UPDATE로 인한 키 추가/삭제 시에도 버퍼링 가능
- innodb_change_buffering 시스템 변수 도입 - 작업별로 활성여부 설정 가능

  ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/84b485e1-4168-43b9-906d-ed0d00fcb4ea/Untitled.png)


### 버퍼 풀 메모리

- 체인지 버퍼는 InnoDB 버퍼 풀 메모리 공간의 25(default)~50% 까지 설정 가능
- innobd_change_buffer_max_size 시스템 변수로 비율 설정
- 메모리 사용량 및 얼마나 많은 사항을 버퍼링 하는지 확인 가능

    ```sql
    -- 체인지 버퍼가 사용중인 메모리 공간 크기
    SELECT EVENT_NAME, CURRENT_NUMBER_OF_BYTRS_USED
      FROM performance_schema.memory_summary_global_by_event_name
     WHERE EVENT_NAME = 'memory/innodb/ibuf0ibuf';
    
    -- 체인지 버퍼 관련 오퍼레이션 처리 횟수
    SHOW ENGINE INNODB STATUS \G;
    ```


# 4.2.11 리두 로그 및 로그 버퍼

: **리두 로그(Redo Log)**는 트랜잭션 4가지 요소 ACID 의 **D(Durable) = 영속성**과 밀접하다.

비정상적인 종료로 인해 데이터 파일에 기록되지 못한 데이터를 잃지 않게 해주는 안전장치 이다.

### WAL(Write Ahead Log) = Redo Log , 데이터를 디스크에 기록하기 전에 기록되는 로그

대부분의 DB Server는 데이터 변경 내용능 로그로 먼저 기록한다.

- 대부분의 DBMS에서 읽기 >>> 쓰기, 읽기 성능을 고려한 자료구조임
- 파일 쓰기의 경우 디스크 랜덤 엑세스가 필요 (상대적으로 큰 비용이 필요)
- 이러한 이유로, 쓰기 비용이 낮은 리두 로그 존재 (비정상 종료시, 리두로그를 이용해 복구)
- 또한, ACID와 더불어 성능을 위해, 리두로그를 버퍼링 하는 InnoDB 버퍼풀 및 로그 버퍼를 위한 자료구조가 존재

### 비정상적인 종료

1. 커밋됐지만, 데이터 파일에 기록되지 않은 데이터

   → 리두 로그에 저장된 데이터를 데이터 파일에 복사

2. 롤백됐지만, 데이터 파일에 이미 기록된 데이터

   → 리두 로그로 해결 불가능. 언두 로그의 데이터를 데이터 파일로 복사

   단, 변경이 커밋/롤백/트랜잭션 실행 중간이 었는지에 대한 확인을 위해 리두로그 확인 필요


### 리두 로그는 트랜잭션 커밋 즉시, 디스크로 기록되도록 시스템 변수 설정 권장

→ 그래야지 비정상적인 종료 복구시, 장애직전 시점까지 복구 가능

- 하지만 이는 많은 부하를 유발

### 1) 주기

- `innodb_flush_log_at_trx_commit`

  ![Untitled](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/eb27bd2d-c944-44df-86bb-f59f94af0995/Untitled.png)

- DDL실행시 리두 로그가 디스크로 동기화

  → 1초 옵션을 주어도 그 사이에 DDL 발생시, 그 보다 짧은 간격이 동기화 될 수 있음

- `innodb_flush_log_at_timeout` : 디스크 동기화 시간 설정가능 (default: 1초) - 잘 변경 안함

### 2) 크기

- `innodb_log_file_size` : 리두 로그 파일 크기
- `innodb_log_files_in_group` : 리두 로그 파일 개수
- 전체 리두 로그 파일 크기 = (`innodb_log_file_size`) * (`innodb_log_files_in_group`)

### 리두로그 버퍼

- 리두 로그 기록시, 기록이 많으면 버퍼링
- 기본값 16MB 수준이 적절
- BLOB, TEXT와 같은 큰 데이터를 자주 변경하는 경우, 더 크게 설정 권장

### 4.2.11.1 리두 로그 아카이빙

MySQL 8.0 부터 리두 로그 아카이빙 기능 추가됨

### 4.2.11.2 리두 로그 활성화 및 비활성화

# 4.2.12 어댑티브 해시 인덱스

# 4.2.13 InnoDB와 MyISAM, MEMORY 스토리지 엔진 비교
