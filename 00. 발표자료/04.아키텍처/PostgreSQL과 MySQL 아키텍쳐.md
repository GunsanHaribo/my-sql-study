# 핵심 차이점

|            | PostgreSQL        | MySQL            |
| ---------- | ----------------- | ---------------- |
| 아키텍쳐 단위    | 프로세스              | 스레드              |
| 스토리지 모델    | 통합 엔진             | 플러그형 엔진(컴포넌트형)   |
| 기본 격리 수준   | READ COMMITTED    | REPEATABLE READ  |
| MVCC 구현 방법 | 각 명령문 시작 시 스냅샷 생성 | 트랜잭션 시작 시 스냅샷 생성 |


# 기본 아키텍쳐
## PostgreSQL - 프로세스
<img width="2356" height="1072" alt="Pasted image 20251015151220" src="https://github.com/user-attachments/assets/470af123-5be7-42ca-9b3f-fd8e80e64146" />
### Postgres 서버
 프로세스(postmaster)
- PostgreSQL 인스턴스가 시작될 때, 가장 먼저 실행되는 마스터 프로세스
- 기본 포트는 5432
	- https://www.postgresql.org/docs/18/app-postgres.html
- 클라이언트 연결 요청을 받고 새 백엔드 프로세스를 생성(fork)함
- 공유메모리 영역을 할당하고 각종 백그라운드 프로세스들을 시작함
	- 필요에 따라 복제 관련 프로세스와 백그라운드 워커 프로세스를 실행함
- 전체 데이터베이스 클러스터를 관리, 감독함
- 데이터베이스 종료 시 모든 프로세스 정상 종료시키는 역할 수행

### 백엔드 프로세스(postgres)
- Postgres 서버 프로세스에 의해 시작되어 연결된 클라이언트 한 명이 보내는 모든 쿼리 처리
- 단일 TCP 연결 사용
- 각 백엔드 프로세스는 한 번에 하나의 DB에서만 작동
	- 연결 중에 명시적으로 데이터베이스를 선택해야함
- 클라이언트로부터 받은 SQL 쿼리를 파싱, 분석, 최적화하여 실행 계획 수립, 실행 결과를 클라이언트에게 반환함

### 백그라운드 프로세스
- 데이터베이스 시스템을 원활하고 안정적으로 유지하기 위해 뒤에서 동작하는 필수 프로세스들
- 종류
	- 백그라운드 Writer
	- 체크포인터
	- Auto Vacuum launcher
	- WAL Writer
	- WAL Summarizer
	- 통계 수집기
	- 로깅 수집기
	- I/O 작업자
	- 보관자

## MySQL - 스레드
<img width="565" height="430" alt="Pasted image 20251015151348" src="https://github.com/user-attachments/assets/9fb19e00-2c57-46d6-a00f-d5f07e1daa10" />

### MySQL 서버 프로세스(mysqld)
- MySQL 서버의 핵심이 되는 단일 멀티스레드 프로세스
	- https://dev.mysql.com/doc/refman/8.4/en/mysqld.html
- 기본 포트는 3306
	- https://dev.mysql.com/doc/refman/8.4/en/selinux-context-mysqld-tcp-port.html
- 서버가 시작될 때 실행되어 전체적인 작업을 관리, 클라이언트 요청을 처리할 스레드를 생성 및 관리함
- 각종 백그라운드 스레드를 제어함

### 포그라운드 스레드(Foreground Threads / User Threads)
- 클라이언트 연결 하나당 하나씩 생성되는 스레드
- 클라이언트로부터 받은 SQL 쿼리를 파싱, 분석, 최적화하여 실행 계획 수립, 실행 결과를 클라이언트에게 반환함

### 백그라운드 스레드
- InnoDB 스토리지 엔진을 비롯한 여러 컴포넌트를 관리하기 위해 백그라운드에서 실행되는 스레드
- 대부분 InnoDB 엔진에 의해 실행됨
- 종류
	- 메인 스레드
	- I/O 스레드
	- 페이지 클리너 스레드
	- Purge 스레드


# 메모리 아키텍쳐
## PostgreSQL
<img width="415" height="336" alt="Pasted image 20251015152948" src="https://github.com/user-attachments/assets/0c668b5a-5be1-4ce3-8b12-a77ed60348d6" />


### 공유 메모리
- PostgreSQL 인스턴스의 모든 프로세스들이 접근하고 공유하는 공간
- 주요 구성 요소
	- Shared Buffers(공유 버퍼)
		- 테이블과 인덱스 내 페이지를 디스크에서 본 영역으로 로드하고 직접 실행함
	- WAL Buffers(WAL 버퍼)
		- 디스크에 쓰기 전 WAL 데이터를 버퍼링하는 영역
	- Commit Log(CLOG)
		- MVCC 메커니즘을 위해 모든 트랜잭션 상태를 보관하는 영역
	- 기타

### 로컬 메모리
- 각 백엔드 프로세스에 개별적으로 할당되는 메모리 공간
- 주요 구성 요소
	- work_mem
		- ORDER BY 및 DISTINCT 연산으로 튜플 정렬하고 병합-조인 및 해시 - 조인 연산으로 테이블 조인함
	- maintenance_work_mem
		- 유지 관리 작업(Vacuum 및 재 색인)
	- temp_buffers
		- 임시 테이블 저장하는데 사용



## MySQL
<img width="406" height="323" alt="Pasted image 20251015151443" src="https://github.com/user-attachments/assets/cb83f94c-5d02-4287-9d2e-c1cc20edfb34" />

### 글로벌 메모리 영역(Global Memory Area)
- mysqld 프로세스가 시작될 때 할당되며, 모든 스레드가 공유함
- 주요 구성 요소
	- InnoDB 버퍼 풀
	- Redo 로그 버퍼
	- 바이너리 로그 버퍼
	- 테이블 캐시

### 세션 메모리 영역(Session / Thread - Specific Memory Area)
- 클라이언트 스레드가 생성될 때 할당되고 연결 종료 시 해제
- 주요 구성 요소
	- 정렬 버퍼(Sort Buffer)
	- 조인 버퍼(Join Buffer)
	- 읽기 버퍼(Read Buffer)

# 출처
https://www.postgresql.org/docs/current/overview.html

https://www.interdb.jp/pg/pgsql02/index.html

# 부록
## 트랜잭션 격리 수준에서 발생하는 문제

Dirty Read
- 아직 커밋되지 않은 다른 트랜잭션의 변경 데이터를 읽는 현상

Non-Repeatable Read
- 한 트랜잭션 내에서 동일한 행을 두 번 읽었는데, 그 사이에 다른 트랜잭션이 해당 행을 수정하고 커밋하는 바람에 두 번의 읽기 결과가 다르게 나타나는 현상

Phantom Read
- 한 트랜잭션 내에서 특정 범위 데이터를 두 번 조회했는데, 첫 조회에서는 없었던 행이 두 번째 조회에서 나타나는 현상
- 다른 트랜잭션에서 수행한 변경 작업에 의해 레코드가 보였다 안 보였다 하는 현상

## 트랜잭션 격리 수준

READ UNCOMMITTED
- 한 트랜잭션이 다른 트랜잭션의 커밋되지 않은 데이터까지 읽을 수 있음


READ COMMITTED
- 커밋된 데이터만 읽을 수 있도록 보장함
- 어떤 트랜잭션에서 데이터를 변경했더라도 COMMIT 완료된 데이터만 트랜잭션에서 조회할 수 있음


REPEATABLE READ
- 트랜잭션이 끝날 때까지 몇 번을 읽어도 항상 동일한 데이터를 볼 수 있도록 보장함


SERIALIZABLE
- 마치 트랜잭션들을 하나씩 순서대로 실행하는 것처럼 동작하게 만듬

