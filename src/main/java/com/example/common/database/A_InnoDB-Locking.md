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


#### Reference
- https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html
- https://www.geeksforgeeks.org/multiple-granularity-locking-in-dbms/
- https://severalnines.com/blog/understanding-lock-granularity-mysql/
- https://copyprogramming.com/howto/multiple-granularity-locking-in-dbms
- https://bako94.tistory.com/157