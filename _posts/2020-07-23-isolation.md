---
layout: post
title: MySql Isolation
tags: [mysql, lock, isolation]
---


현재 회사에서 MySQL을 사용중이어서 학습을 위해 Real MySQL 책을 읽고 레퍼런스 문서도 확인하면서 그에 Isolation Levels에 따른 동작을 정리하고자 글을 씁니다. 

Isolation Levels 하면 흔히 아래 네가지를 언급하곤 합니다. 

- READ UNCOMMITTED

- READ COMMITTED

- REPEATABLE READ

- SERIALIZABLE


각 Isolation Level 별 동작을 알아보겠습니다. 

먼저 테스트를 진행한 환경은 MySQL 5.7 버전, 기본 설정은 MySql의 기본 Isolation Level인 REPEATABLE READ 입니다.


Isolation Level에 대해 알아보기에 앞서 사전에 알아야 될 부분으로 MySQL에서는 SERIALIZABLE Level을 제외하고 명시적으로 Lock을 선언한 SELECT 문이 아니고서는 어떠한 Lock도 걸지 않는다는 것입니다. 

(명시적으로 FOR UPDATE, LOCK IN SHARE MODE 에서는 Isolation Level 마다 다른 방식으로 Lock을 겁니다. 이 부분은 아래에 설명을 이어나가겠습니다.)


추가로 MySQL에서는 모든 row에 히든 컬럼이 존재하고 해당 컬럼에는 트랜젝션 ID가 기록되어 있다는 점과 삭제의 경우에도 COMMIT 전 해당 row에 삭제 FLAG를 설정하고 UPDATE와 유사하게 처리하고 

Rollback Segment 안에는 INSERT / UPDATE + DELETE 를 위한 두개의 UNDO 영역이 존재한다는 것 입니다. 



그럼 위에 Isolation Level에 대한 설명을 다시 이어나가겠습니다. 

  
테이블은 아래와 같이 생성되어있고 데이터는 기본적으로 3건을 삽입하였습니다. 

```sql
CREATE TABLE employee (
    emp_no int (10),
    name varchar(100),
    primary key (emp_no)
);

INSERT employee VALUES (1, 'user1');
INSERT employee VALUES (2, 'user2');
INSERT employee VALUES (3, 'user3');

```


첫번째로, READ UNCOMMITTED 의 경우 흔히 나오는 문제로 DIRTY READ가 발생한다고 하는데요. 

이 DIRTY READ가 무엇인가하면 세션A, 세션B가 동시에 데이터에 접근하여 작업을 할 때 발생하는 현상 입니다. 

먼저 세션A가 트랜젝션을 시작합니다. 이때 트랜젝션 ID는 1입니다.

```sql
START TRANSACTION;

SELECT name FROM employee;
```

위 SELECT 문을 실행했을 때 데이터는 user1, user2, user3 세개가 나옵니다. 

이상태로 동시에 세션B가 트랜젝션을 시작합니다. 이때 세션A의 트랜젹센은 아직 COMMIT 또는 ROLLBACK 되지 않은 상태 입니다.

```sql
START TRANSACTION; 

INSERT employee VALUES (4, 'user4');
```

위와 같이 INSERT 문으로 4번째 데이터를 삽입합니다. 

이후 다시 세션A가 처음 실행했던 SELECT 문을 다시 실행했을 경우 어떻게 동작할까요? 

```sql
SELECT name FROM employee;
```

직접 실행해보면 결과를 알 수 있듯이 user1, user2, user3, user4가 나옵니다. 

결과와 같이 아직 세션B의 트랜젝션이 COMMIT되지 않았는데 세션A에서 등장하게 됩니다. 이 COMMIT 되지 않았는데 나오는 데이터를 DIRTY READ 라고 합니다. 


두번째로, READ COMMITTED 입니다. 해당 Isolation level에서는 NON-REPEATABLE, PHANTOME READ가 발생한다고 합니다. 

NON-REPEATABLE이 발생하는 원인을 예를 들어 설명해보겠습니다. 테이블 및 데이터는 위 예시와 동일합니다. 

먼저 세션A가 트랜젝션을 시작합니다. 이때 트랜젝션 ID는 1입니다.

```sql
START TRANSACTION;

SELECT name FROM employee;
```

다음으로 세션B가 트랜젝션을 시작합니다. 이때 트랜젝션 ID는 2고 이번에는 첫번째와 같이 데이터를 삽입해보겠습니다.

```sql
START TRANSACTION; 

INSERT employee VALUES (4, 'user4');

COMMIT;
```

이후 다시 세션A가 처음과 같은 SELECT 문으로 데이터를 조회합니다. 

```sql
SELECT name FROM employee;
```

첫번째 READ UNCOMMITTED의 경우에는 COMMIT 되지 않은 데이터가 나온게 문제였다면 READ COMMIITED의 경우에는 COMMIT 된 데이터로 인해서 

user1, user2, user3, user4가 나오게 됩니다. 

이를 PHANTOME READ가 발생했다 합니다. 즉 트랜젝션 내에서 동일한 SELECT을 실행 시 없던 데이터가 생기는 현상을 말합니다. 



이번에는 위 내용을 적용하지 않았다는 가정하에 이번에는 UPDATE를 기준으로 설명해보겠습니다. 

세션A가 트랜젝션을 시작합니다. 

```sql
START TRANSACTION;

SELECT name FROM employee WHERE emp_no = 1;
```

다음으로 세션B가 트랜젝션을 시작하고 

```sql
START TRANSACTION; 

UPDATE employee SET name = 'change_user1' WHERE emp_no = 1;

COMMIT;
```

그리고 다시 세션A가 처음과 같은 SELECT 문으로 조회합니다. 

```sql
START TRANSACTION;

SELECT name FROM employee WHERE emp_no = 1;
```

결과를 보면 name은 change_user1이 나옵니다. 세션A의 트랜젝션이 진행중인데도 세션B의 트랜젝션의 영향으로 데이터가 일관되게 나오지 않는 현상이 발생한 것 입니다.

이를 NON-REPEATABLE READ라 합니다. 


그럼 이러한 문제들을 해결하기 위한 방법이 뭐가 있을까요? 


세번째로, REPEATABLE READ 입니다. 

이 부분을 설명하기 전 알아야 할 부분으로 LOCK 종류가 있는데, MySQL의 경우 LOCK은 실제 데이터가 아닌 INDEX에 걸게됩니다. 이 INDEX에 락을 거는 방식으로 

ROW INDEX LOCK, GAP LOCK, NEXT KEY LOCK 세 종류가 있는데 순서대로 간략하게 설명하면 

ROW INDEX LOCK은 조건에 일치하는 INDEX ROW에 거는 LOCK, GAP LOCK은 조건에 일치하는 INDEX ROW의 간격에 거는 LOCK, NEXT KEY LOCK은 ROW INDEX LOCK과 GAP LOCK의 조합입니다. 


다시 REPEATABLE READ에 대한 설명으로 이어가겠습니다. 진행 순서는 이전과 동일합니다. 

세션A가 트랜젝션을 시작합니다. 

```sql
START TRANSACTION;

SELECT name FROM employee WHERE emp_no = 1;
```

다음으로 세션B가 트랜젝션을 시작하고 

```sql
START TRANSACTION; 

UPDATE employee SET name = 'change_user1' WHERE emp_no = 1;

COMMIT;
```

그리고 다시 세션A가 처음과 같은 SELECT 문으로 조회합니다. 

```sql
START TRANSACTION;

SELECT name FROM employee WHERE emp_no = 1;
```

이때 결과는 어떻게 나올까요? 직접 쿼리를 실행해보면 알 수 있듯이 name 은 여전히 수정되기 전인 user1이 나오게 됩니다. 이건 어떻게 가능할까요? 

조금 더 상세하게 이 부분의 동작 순서를 확인해보면 세션A가 트랜젝션을 시작함과 동시에 트랜젝션 ID를 할당받게 됩니다. 순서대로 이 값이 1이라고 하겠습니다. 

다음으로 세션2의 트랜젝션이 시작함에 따라 트랜젝션 ID 2를 할당받게 됩니다. 


REPEATABLE READ에서는 이 트랜젝션 ID를 통해 일관성 있는 데이터를 조회하게 됩니다. 

데이터가 변경되는 작업인 INSERT, DELETE, UPDATE 시에 흔히 스냅샷이라고 표현되는 UNDO 에 데이터가 삽입됩니다. 이 데이터 역시 당시 트랜젝션의 ID를 삽입하게 되는데요.


세션A의 두번째 SELECT 시 첫번째 SELECT와 동일한 데이터를 가져올 수 있는 비법은 여기에 있습니다. 

조회 시 현재 트랜젝션 ID보다 이전 데이터만을 가져오게 되는 것이죠. 


세션B에서 emp_no 가 1인 ROW는 데이터를 수정 시 UNDO에 스냅샷을 남기게 됩니다. 이때 UPDATE이기에 변경된 컬럼의 이전 데이터와 트랜젝션 ID를 스냅샷으로 남깁니다. 

그리고 수정된 ROW에는 데이터 수정과 동시에 현재 트랜젝션 ID인 2가 기록되게 됩니다. 


이후 세션A가 데이터를 조회했을 때 현재 트랜젝션 ID보다 높은 ROW인 emp_no 가 1인 ROW는 UNDO의 스냅샷중 현재 트랜젝션 ID (1) 보다 작거나 같은 트랜젝션 ID의 스냅샷을 골라 

가져오게 됩니다. 이런 과정을 통해 세션A에서는 트랜젝션 내에서는 일관된 데이터를 가져오게 되는 것이죠. 


이런 과정은 INSERT 시에도 동일합니다. UPDATE와 같은 원리로 새로 추가된 데이터라도 트랜젝션 ID 비교를 통해 일관되게 데이터를 가져올 수 있게 되는 것이죠. 


이전에 LOCK에 대해 설명했으니 이번에는 조금 다른 내용을 추가해보도록 하겠습니다. 


LOCK을 통해 SELECT 시 READ COMMITTED와 REPEATABLE READ에 대한 비교 입니다. 


먼저 READ COMMITTED 입니다. 

세션A가 트랜젝션을 시작합니다. 그리고 SELECT 문에서 FOR UPDATE를 통해 배타적 LOCK을 겁니다. 


```sql
START TRANSACTION;

SELECT name FROM employee WHERE emp_no > 1 FOR UPDATE;
```

이후 세션B가 트랜젝션을 시작하고 이번에는 INSERT 문을 통해 데이터를 추가합니다.

```sql
START TRANSACTION; 

INSERT employee VALUES (4, 'user4');

COMMIT;
```

이 INSERT문은 정상적으로 실행될 것 입니다. 그리고 다시 아래와 같은 SELECT 문을 세션A가 실행합니다.

```sql
SELECT name FROM employee WHERE emp_no > 1 FOR UPDATE;
```

이때 결과는 어떤 값이 나올까요? 

실제 실행해보면 나오듯이 user1, user2, user3, user4가 나옵니다. SELECT ... FOR UPDATE를 통해 LOCK을 걸 경우 애초에 세션B에서 INSERT문이 실행이 안될거라고

생각할 수 있는데 막상 실행을 해보면 문제 없이 INSERT문은 동작하게 됩니다. 


이 INSERT문 동작에 문제가 없는 이유는 READ COMMITTED는 LOCK을 통한 SELECT 시 조건에 일치하는 ROW에만 LOCK 을 걸게 됩니다. 

결국 emp_no 가 2, 3인 INDEX ROW에만 LOCK이 걸리는 것이죠.


그래서 세션B의 INSERT문은 문제 없이 동작할 수 있던 것 입니다. 그럼 REPEATABLE READ일 경우는 어떨까요?

위와 동일한 시나리오로 실행 시 REPEATABLE READ의 경우 세션B에서 INSERT 를 할 수 없습니다. 그에 대한 이유는 REPEATABLE READ일 경우 LOCK 을 사용하는 SELECT는 NEXT KEY LOCK을 사용하게 됩니다. 


이게 어떤 마법을 부리기에 INSERT가 되지 않는가 하면 emp > 1 이라는 조건에 대해 emp_no 2, 3 그리고 이후의 GAP(3이상 범위)에 대해서 LOCK을 걸게되고 그 LOCK으로 인해 세션B는 INSERT를

할 수 없게 되는 것이죠. 이 내용으로만 보면 REPEATABLE READ에서는 NON-REPEATABLE, PHANTOM READ 모두 발생을 안할 것 같습니다. 

*일반적*으로는 발생 안하는 것이 맞지만 항상 그렇지는 않습니다. 


그에 대한 예를 들어볼까요? 다시 한번 초기 데이터를 기준으로 아래 순서대로 시나리오가 흘러간다는 가정하에 설명 드리겠습니다. 

먼저 세션A가 트랜젝션을 시작하고 SELECT문을 실행합니다. 

```sql
START TRANSACTION;

SELECT name FROM employee WHERE emp_no > 1;
```

이후 세션B가 트랜젝션을 시작하고 이번에는 INSERT 문을 통해 데이터를 추가합니다.

```sql
START TRANSACTION; 

INSERT employee VALUES (4, 'user4');

COMMIT;
```

이 INSERT문은 정상적으로 실행될 것 입니다. 그리고 다시 아래와 같은 SELECT 문을 세션A가 실행합니다.

```sql
SELECT name FROM employee WHERE emp_no > 1 FOR UPDATE;
```

이 예제는 Real MySQL에 나온 예제로 예제에서는 세션A에서 FOR UPDATE 를 통해 SELECT 를 배타적 LOCK을 걸고 실행하였지만 실제 테스트 결과 책의 내용과 일치하지 않았기에

조금 수정하여 재현 하였습니다. 


위 결과는 어떻게 나올거 같나요? 일단 세션A의 트랜젝션 ID가 1, 세션B의 트랜젝션 ID가 2라는 가정하에 위에서 한 설명대로라면 트랜젝션 ID 값이 2로 들어간 ROW의 경우 세션A에서는

조회가 안될 것이라 생각했는데 예상과는 다른 결과로 user1, user2, user3, user4 였습니다.


왜 이런 현상이 발생한 것일까요? 이런 현상은 FOR UPDATE로 인해 발생한 현상입니다. FOR UPDATE를 사용한 세션A의 두번째 SELECT는 조회 시 트랜젝션ID를 기준으로 조회를 하게됩니다. 

이때 세션B에서 넣은 데이터는 트랜젝션ID 비교 중 SELECT 대상이 아니라 판단하고 UNDO에서 가져오려 하지만 이 UNDO의 데이터는 LOCK을 걸 수 없습니다. (LOCK은 실제 데이터에만 걸 수 있습니다.)

그에 따라 UNDO에 LOCK을 걸지 못하면서 실제 트랜젝션ID가 높은 데이터를 조회하게 되면서 PHANTOM READ가 발생하게 된 것입니다. 


이러한 현상은 일반적인 케이스는 아니지만 충분히 발생할 수 있는 상황으로 비슷하게 PHANTOM READ가 발생하는 다른 경우는 아래와 같습니다. 


먼저 세션A가 트랜젝션을 시작하고 SELECT문을 실행합니다. 

```sql
START TRANSACTION;

SELECT name FROM employee WHERE emp_no > 1;
```

이후 세션B가 트랜젝션을 시작하고 이번에는 INSERT 문을 통해 데이터를 추가합니다.

```sql
START TRANSACTION; 

INSERT employee VALUES (4, 'user4');

COMMIT;
```

이 INSERT문은 정상적으로 실행될 것 입니다. 그리고 다시 아래와 같은 UPDATE문을 세션A가 실행합니다. 

```sql
UPDATE employee SET name = 'change_user' WHERE emp_no > 1;
```

이후 다시 세션B에서 첫번째와 동일한 SELECT 문을 실행 합니다. 

```sql
START TRANSACTION;

SELECT name FROM employee WHERE emp_no > 1;
```

이 경우에도 역시 user1, change_user, change_user, change_user 총 4건의 데이터가 조회될 것 입니다. 이 부분은 아마 세션A의 UPDATE문으로 인해 세션B에서 넣은 데이터의 

최종 변경 트랜젝션ID가 세션A의 트랜젝션ID로 변경되게 되면서 발생하는 이상 현상으로 보입니다. 


이런 DIRTY READ, NON REPEATABLE, PHANTOM READ와 같은 이슈들이 발생하지 않는 Isolataion Level이 SERIALIZABLE READ입니다. 


SERIALIZABLE READ 는 기본적으로 SELECT 에 LOCK을 명시하지 않아도 LOCK IN SHARE MODE 로 SELECT 가 실행됩니다. 

그에 따라 NEXT KEY LOCK 을 통해 INDEX ROW와 GAP에 모두 항상 LOCK을 걸기에 모든 이상 현상은 발생하지 않게 됩니다.





