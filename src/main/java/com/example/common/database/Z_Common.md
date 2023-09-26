#### [REPEATABLE READ](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_repeatable_read)
`InnoDB`의 디폴트 격리수준(`isolatation level`)이다.  
이 격리수준에서는 `NON REPEATABLE READ`를 막아주지만 `PHANTOM READ`는 막아주지 못한다.  
트렌젝션이 시작될 떄에 스냅샷을 생성하고 트렌젝션이 끝날때까지 모든 쿼리가 해당 스냅샷을 보고 수행된다.  
아래와 같은 명령어를 수행하면 다른 트렌젝션은 대기하게 된다.  
`UPDATE/DELETE ... WHERE`, `SELECT ... FOR UPDATE`, `LOCK IN SHARE MODE`, `SELECT ... FOR SHARE`(`SELECT ... LOCK IN SHARE MODE`)

#### [locking read](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_locking)
`SELECT` 구문이지만 `InnoDB`에서 락을 걸기도 한다. 이를 `locking read`라 한다.  
이는 `SELECT` 가 `S LOCK`은 아니라는 의미이기도 하다.  
`SELECT ... FOR UPDATE`, `SELECT .. LOCK IN SHARE MODE`, `SELECT ... FOR SHARE`(`SELECT ... LOCK IN SHARE MODE`)등이 예시이다.  
이 쿼리들은 수행되는 트렌젝션의 격리수준(`isolatation level`)에 따라 `deadlock`을 유발할 수도 있다.

#### [undo log](https://dev.mysql.com/doc/refman/8.0/en/glossary.html#glos_undo_log)
실행중인 트렌젝션에 의해서 데이터가 변경되면 변경되기 전의 원본은 `undo log`에 보관한다.  
트렌젝션이 커밋되면 더이상 원본은 필요없기 때문에 이 `undo log`는 없어질 것이고, 롤백된다면 이 `undo log`를 이용해서 원본으로 복구할 것이다.  

#### [Clustered and Secondary Indexes](https://dev.mysql.com/doc/refman/8.0/en/innodb-index-types.html)
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