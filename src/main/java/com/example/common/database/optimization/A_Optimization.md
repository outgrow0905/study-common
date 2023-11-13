#### 전체 컬럼을 조회하는 것과 꼭 필요한 컬럼만 조회하는 것의 차이가 있을까?  
해답은 `mysql`의 정렬 알고리즘에 있다.  
아래의 쿼리를 예시로 시작해보자.
~~~sql
-- pk: id
-- secondary index 없음
SELECT id, first_name, last_name FROM employees ORDER BY last_name;
~~~

위의 쿼리는 두 가지 방식으로 처리될 수 있다.
~~~
- single pass: pk와 정렬에 필요한 last_name 두 가지만 일단 가져와서 레코드를 정렬한다. 정렬 이후에 다시 pk로 데이터를 조회하여 작업을 마무리한다.
- tww pass: 모든 데이터를 전부 가져와서 정렬한다. 추가 조회작업은 없다. 
~~~
둘의 작업은 장단점이 있을 것이다.  
`single pass`는 더 많은 레코드를 `소트버퍼`에 가져와서 정렬할 수 있지만, 조회를 한번 더 해야하는 단점이 있다.  
`two pass`는 더 적은 레코드를 `소트버퍼`에 올릴 수 밖에 없지만, 추가 조회를 할 필요는 없다.  
최신 버전에서는 특정 조건을 제외하고는 일반적으로 `single pass`를 사용하게 된다. 
여기서 특정조건이라 함은 레코드 타입이 `BLOB`, `TEXT`와 같은 경우이다.  

위에서 언급했듯 일반적으로 `single pass`가 사용되기 때문에 `소트버퍼`에 최대한 많은 레코드를 가져올수록 이득이다.  
따라서, `two pass`가 수행되는 조건이 아니라면 꼭 필요한 레코드만 조회하는 것이 더 빠를 수 있다.  



#### ORDER BY 최적화 (정렬처리방법)
mysql의 정렬처리방법은 세 가지로 처리될 수 있다.  
처리속도는 순서대로 느려진다.  

~~~
- 인덱스를 사용한 정렬
- 드라이빙 테이블만 먼저 정렬하고 조인 수행
- 모든 조인과 쿼리 수행을 마치고 결과값을 임시테이블에 저장 후 정렬
~~~

##### 인덱스를 사용한 정렬
~~~sql
SELECT *
  FROM employees a, salaries s
 WHERE a.emp_no = s.emp_no
   AND a.emp_no BETWEEN 100002 AND 200020 -- pk
ORDER BY e.emp_no;
~~~
`employees` 테이블의 `pk`가 `emp_no` 이므로 여기서는 인덱스를 사용한 정렬이 수행된다.  
수행될 수 있는 이유는 `WHERE` 조건과 `ORDER BY` 조건에서 같은 인덱스(`pk`)를 사용할 수 있기 때문이다.

##### 드라이빙 테이블만 먼저 정렬하고 조인 수행
~~~sql
SELECT *
  FROM employees a, salaries s
 WHERE a.emp_no = s.emp_no
   AND a.emp_no BETWEEN 100002 AND 200020 -- pk
ORDER BY e.last_name;
~~~
위의 쿼리에서 `ORDER BY` 조건만 변경한 예시이다.  
이러한 경우에는 먼저 `employees` 테이블에서 `pk`로 조회를 마치고 `소트버퍼`에서 `employees` 테이블을 `last_name` 조건으로 정렬을 먼저한다. 
그렇게 수행되는 이유는 일단 드라이빙 테이블로 `employees` 테이블이 선택되고, `ORDER BY`의 조건이 전부 드라이빙 테이블에 속한 컬럼들이기 때문이다.
조인과 쿼리수행을 마친 거대한 데이터를 `소트버퍼`에서 정렬하는 것보다는 부담이 적을 것이다.  

이러한 수행의 쿼리플랜에는 `Extra` 컬럼에 `Using filesort`가 표시된다.

##### 모든 조인과 쿼리 수행을 마치고 결과값을 임시테이블에 저장 후 정렬
~~~sql
SELECT *
FROM employees a, salaries s
WHERE a.emp_no = s.emp_no
  AND a.emp_no BETWEEN 100002 AND 200020 -- pk
ORDER BY s.salary;
~~~
위의 예시에서 `ORDER BY` 조건만 변경하였다.  
여기서 `ORDER BY` 조건이 salaries 테이블의 컬럼이므로 드라이빙 테이블만 먼저 정렬한 뒤에 조인을 수행하기는 어렵다.  
어쩔 수 없는 경우로 조인과 쿼리수행을 모두 마친뒤에 정렬버퍼에서 정리하는 수밖에 없다.

이러한 수행의 쿼리플랜에는 `Extra` 컬럼에 `Using temporary; Using filesort`가 표시된다.



##### ORDER BY와 LIMIT
위의 세가지 정렬방식중에서 첫번째 방식인 인덱스를 사용한 정렬을 제외하고는 `LIMIT`을 거는 것이 속도개선에 크게 영향을 미치지 못한다.  
인덱스를 타지 못하기 때문에 `LIMIT`은 중간부터 일부분을 요청하더라도 서버에서는 처음부터 읽고 정렬을 해야하기 때문이다.   

연습용으로 각 정렬방법에 대한 읽어야할 데이터 건수를 직접 비교해보자.

~~~sql
-- tb_test1이 드라이빙 테이블인 경우
-- tb_test1 100개의 데이터
-- tb_test2 1000개의 데이터
SELECT *
  FROM tb_test1 t1, tb_test2 t2
 WHERE t1.col1 = t2.col1
 ORDER BY t1.col2
 LIMIT 10;
~~~

~~~
- 인덱스 정렬: t1.col2에 인덱스가 있는 경우일 것이다. t1에서는 1개의 데이터를 읽고, 해당 데이터로 t2와 조인을 시도한다(Nested Loop). t2에서는 1*10개의 데이터를 읽을 것이다. 추가 정렬작업은 필요없다.
- 드라이빙 테이블 먼저 정렬: t1.col2에 인덱스가 없는 경우일 것이다. t1에서 100개의 데이터를 전부 읽고 정렬한다. 정렬 후에 t1의 데이터 하나씩 t2와 조인을 시도한다. t2에서는 1*10개의 데이터를 읽고 조인은 종료된다. LIMIT 10이라 조인은 1번이면 된다.
- 임시테이블 정렬: ORDER BY조건이 t2.col2이라고 가정해보자. t1에서 100개의 데이터를 전부 읽고, 100번의 조인을 전부 수행하여 t2의 100*10개의 데이터를 전부 읽는다. 그리고 100*10개의 데이터를 t2.col2로 정렬을 한다. 
~~~

~~~sql
-- tb_test2이 드라이빙 테이블인 경우
-- tb_test1 100개의 데이터
-- tb_test2 1000개의 데이터
SELECT *
  FROM tb_test1 t1, tb_test2 t2
 WHERE t1.col1 = t2.col1
 ORDER BY t2.col2
 LIMIT 10;
~~~

~~~
- 인덱스 정렬: t2.col2에 인덱스가 있는 경우일 것이다. t2에서는 1개의 데이터를 읽고, 해당 데이터로 t1와 조인을 시도한다(Nested Loop). 총 10번의 조인을 해야한다. t2, t1에서 각각 10개의 데이터를 읽게 된다. 정렬작업은 사실상 없다.
- 드라이빙 테이블 먼저 정렬: t2.col2에 인덱스가 없는 경우일 것이다. t2에서 1000개의 데이터를 전부 읽고 정렬한다. 정렬이후 하나씩 t1과 조인을 시도한다. 총 10번의 조인을 해야한다. t2에서 1000개의 데이터를 읽었고, t1에서는 10개의 데이터를 읽게 된다.  
- 임시테이블 정렬: ORDER BY조건이 t1.col2이라고 가정해보자. t2에서 1000개의 데이터를 전부 읽고, 1000번의 조인을 전부 수행하여 t1의 100개의 데이터를 전부 읽는다. 그리고 t1.col2로 1000개 데이터를 정렬한다. 
~~~


#### References
- https://dev.mysql.com/doc/refman/8.0/en/group-by-optimization.html