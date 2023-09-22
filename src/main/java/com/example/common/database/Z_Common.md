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

