#### Intention Locks
`intention lock`은 테이블에 설정하는 락이다.  
`intention shared lock(IS)`, `intention exclusive lock(IX)` 두 종료가 있다.    
테이블에 설정함과 동시에 하위 레벨인 `row` 단위에도 락을 설정하게 된다.    

`IS lock`는 `row`에 `S lock`을 설정하게 된다.  
예를 들어 `SELECT ... FOR SHARE` 구문은 `테이블`에는 `IS lock`을 설정하고 설정범위가 되는 `row`에는 `S lock`을 설정한다.

`IX lock`은 `row`에 `X lock`을 설정하게 된다.  
예를 들어 `SELECT ... FOR UPDATE` 구문은 `테이블`에는 `IX lock`을 설정하고 설정범위가 되는 `row`에는 `X lock`을 설정한다.   
(`UPDATE, INSERT, DELETE`도 당연히 같다.)

호환되는 표는 아래와 같다.

 |    | X    | IX    | S     | IS|
|-----|-------|-------|---|---|
| X   |Conflict| Conflict | Conflict  |Conflict  |
| IX  |Conflict| Compatible | Conflict  |Compatible  |
| S   |Conflict| Conflict | Compatible | Compatible |
| IS  |Conflict| Compatible | Compatible | Compatible |

위의 표에서 `X, IS, S, IS` 컬럼들은 모두 `테이블`에 걸리는 락이라고 이해하고 시작하자.  
또한 `Conflict`라고 해서 트렌젝션이 실패하는 것을 의미하는 것은 아니다. 일단 대기하게 되는 상태라고 생각해야 한다.  
`Compatible`이라면 `테이블`단위 `lock` 획득 과정은 통과한다고 이해해야 한다. `테이블` 단위 `lock` 획득 과정을 통과해도 `row` 단위 `lock`을 획득하는 과정에서 대기하게 될 수 있다.

하나씩 설명해보겠다.  

`t1`에 `IX lock`이 걸린 상황부터 보자.  
일단 특정 `테이블`에 `IX lock`을 하는 방법은 많지만 특정 `row`에 `update`문을 수행했다고 가정해보자.

`t1.IX - t2.X`: `t1`에서 특정 `테이블`의 특정 `row`에 `X lock`을 설정하는 작업이 수행중인데, 같은 `테이블`에 `X lock`을 수행할 수는 없을 것이다.  
`t1.IX - t2.IX`: `t1`에서 특정 `테이블`의 특정 `row`에 `X lock`을 설정하는 작업이 수행중인데, 같은 `row`가 아니라면 `테이블` 단위 `lock` 경합부터 굳이 막을 필요는 없을 것이다. 
혹시 같은 `row`를 수정하려고 한다면 해당 `row`에 `X lock`을 획득할 때까지 `t2`는 대기해야 한다.
`t1.IX - t2.S`: `t1`에서 특정 `테이블`의 특정 `row`에 `X lock`을 설정하는 작업이 수행중인데, 해당 테이블에 `S lock` (ex. `read only`)를 설정할 수는 없을 것이다.  
`t1.IX - t2.IS`: `t1`에서 특정 `테이블`의 특정 `row`에 `X lock`을 설정하는 작업이 수행중인데, 같은 `row`가 아니라면 `테이블` 단위 `lock` 경합부터 굳이 막을 필요는 없을 것이다. 
혹시 같은 `row`에 `S lock`을 설정하려고 한다면 `S lock`획득까지 기다리면 된다.  


`t1`에 `IS lock`이 걸린 상황을 보자.  
`t1.IS - t2.X`: 특정 `row`에 `S lock`이 걸려있는 상황일 텐데 해당 `테이블`에 `X lock`을 걸수는 없다.    
`t1.IS - t2.IX`: 같은 `row`만 아니라면 막을 필요는 없다. 같은 `row`라면 `t2`가 대기하게 된다.  
`t1.IS - t2.S`: 특정 `row`에 `S lock`을 설정하고 있는데 `테이블` 전체에 `S lock`을 설정하려고 한다니 굳이 막을 필요는 없다.  
`t1.IS - t1.IS`: `S lock`끼리는 호환이 되니 같은 데이터든 아니든 막을 필요가 없다.  



#### Record Locks
`record lock`은 인덱스에 설정하는 `lock`이다.   
`SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE;` 쿼리를 예로 들면 `c1 = 10`에 해당하는 `row`에 `record lock`이 설정된다.  
인덱스에 설정한다고 했는데 인덱스가 없는 테이블이면 어떨까?  
개발자가 직접 생성한 인덱스가 없다 하더라도 `InnoDB`는 `clustered index`를 테이블마다 가지고 있다. `primary key`라고 보아도 무방하다.   
그러한 테이블이라면 `record lock`은 `(hidden) clustered index`에 설정된다.



#### Clustered and Secondary Indexes
모든 `InnoDB` 테이블은 `clustered index`를 가지고 있다. 개발자가 아무런 인덱스를 생성하지 않은 테이블이라 할지라도 말이다.  
`clustered index`는 `primary key`와 동의어로 보아도 무방하다.  
`clustered index`는 일반적인 조회나 `DML` 수행시에 최적화를 위해 사용된다. 

`clustered index`의 생성부터 알아보자.  

`clustered index`는 개발자가 테이블 생성시에 `primary key`를 명시한다면 이를 그대로 `clustered index`로 사용한다.  
`primary key`를 명시하지 않고 `auto-increment`를 명시했다면 해당 컬럼으로 `clustered index`가 생성된다.  
`primary key`도 없고 `auto-increment`를 설정한 컬럼도 없지만 `unique` 인덱스가 있다면 해당 인덱스로 `clustered index`를 생성한다.   
정말 아무것도 없다면 `InnoDB`는 `GEN_CLUST_INDEX`라는 `hidden clustered index`를 생성한다.  
아무것도 없는 테이블은 `InnoDB`가 `6바이트`짜리 `auto-increment` 성격의 `row id` 컬럼을 생성하고 이를 기반으로 `GEN_CLUST_INDEX`를 생성하는 것이다.  
이 컬럼은 새로운 `row`가 들어올 때마다 `InnoDB`가 자동부여하고 당연히 물리적으로 생성된 순서로 정렬할 수 있다.  

`clustered index`를 제외하고 개발자가 생성한 모든 인덱스를 `secondary index`라 한다.  
`secondary index`는 개발자가 설정한 컬럼을 포함하고, 해당 인덱스에 `primary key` 컬럼이 없다면 `primary key`도 자동으로 포함하게 된다.  
`secondary index` 조회시에 `primary key`를 알게되고 이를 사용하여 `clustered index`에서 `row`를 찾는다.


#### Reference
- https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html
- https://www.geeksforgeeks.org/multiple-granularity-locking-in-dbms/
- https://severalnines.com/blog/understanding-lock-granularity-mysql/
- https://copyprogramming.com/howto/multiple-granularity-locking-in-dbms
- https://bako94.tistory.com/157
- https://dev.mysql.com/doc/refman/8.0/en/innodb-index-types.html