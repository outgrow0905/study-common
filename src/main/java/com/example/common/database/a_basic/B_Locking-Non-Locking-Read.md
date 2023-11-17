#### Consistent Nonlocking Reads
`Consistent Nonlocking Reads`를 직역하자면 `일관된` `락을 걸지않는` `읽기`일 것이다.  
`일관된`의 의미는 같은 트렌젝션 내에서 같은 `SELECT`를 수행하면 같은 결과를 보여주는 의미이다.  
`락을 걸지 않는`의 의미는 읽기에 락을 걸지 않아도 일관된 결과를 보여준다는 의미이다.   
예를 들어, `SELECT FOR UPDATE`, `SELECT FOR SHARE`등을 하지 않아도 일관된 결과를 보여준다는 것이다.  
`읽기`는 이 모든 보장은 `읽기`에 국한된다는 의미이다. 이는 후술하겠다.  

`InnoDB`의 기본 `isolataion level`은 `REPEATABLE READ`이다. 이는 기본적으로 `phantom read`를 막지 못한다.  
하지만 `InnoDB`는 `Consistent Nonlocking Reads`를 이용하여 `phantom read`를 일부 방지한다.  

이는 조회시에 조회시점의 스냅샷을 생성하고, 해당 트렌젝션이 종료될 때까지 해당 스냅샷을 이용하기에 가능하다.  
예를 들어, `select * from tb_user where id >= 81;`의 쿼리가 수행된 후,    
다른 트렌젝션에서 `id = 83`인 데이터를 `insert`하더라도 `phantom read`가 발생하지 않는 것이다.   
`select * from tb_user where id >= 81;`의 쿼리가 수행된 시점에 스냅샷을 생성하고 이후 같은 읽기는 해당 스냅샷을 이용하기 때문이다.  

이제 `일관된` `락을 걸지않는` `읽기`에서 `읽기`부분을 살펴보자.  
이는 같은 트렌젝션 내에서 `SELECT`는 일관된 읽기를 보장할 수 있지만, 만약 다른 트렌젝션에서 `DELETE`, `UPDATE` 등이 발생하면 영향을 받는다는 의미이다. 
예시로 살펴보자.  

~~~sql
SELECT COUNT(c1) FROM t1 WHERE c1 = 'xyz'; -- count 0
DELETE FROM t1 WHERE c1 = 'xyz'; -- several rows deleted
~~~
첫번째 쿼리를 수행하고 같은 조건의 두번째 쿼리(`DELETE`)를 수행하면 당연히 지워지는게 없을 것으로 기대할 수 있다.
하지만 만약 첫번째 쿼리와 두번째 쿼리 사이에 다른 트렌젝션에서 `UPDATE`를 통해 `DELETE` 조건에 맞도록 수정되었다면 해당 데이터는 삭제된다.   

또 다른 예시도 보자.  
~~~sql
SELECT COUNT(c2) FROM t1 WHERE c2 = 'abc'; -- count 0
UPDATE t1 SET c2 = 'cba' WHERE c2 = 'abc'; -- several rows updated
~~~
첫번째 쿼리에서 조회되는 것이 없으므로, 두번째 쿼리를 수행하면 UPDATE되는 데이터가 없을 것으로 기대할 수 있다.  
하지만 만약 첫번째 쿼리와 두번째 쿼리 사이에 다른 트렌젝션에서 `UPDATE`를 통해 두번째 쿼리 조건에 맞도록 수정되었다면 해당 데이터는 수정된다.  

추가로, 스냅샷을 생성하는 시점은 `isolataion level`마다 다르다.  
`REPEATABLE READ`조건에서는 최초의 `SELECT` 시점에 생성하고 트렌젝션이 종료될때까지 스냅샷을 다시 바꾸는 일은 없다.  
`READ COMMITED` 조건에서는 `SELECT` 시점마다 스냅샷을 새로 생성한다. 따라서 `phantom read`가 발생할 수 있다.

#### Reference
- https://dev.mysql.com/doc/refman/8.0/en/innodb-consistent-read.html