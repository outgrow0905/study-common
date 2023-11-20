#### 쿼리플랜 읽는 순서
자주 읽어야하는 쿼리플랜을 우리는 잘 읽고있을까?  
mysql 8버전부터는 `explain analyze`를 통해 
실제 쿼리플랜의 어떤 부분에서 얼만큼 걸리는지 수행시간도 같이 확인해볼 수 있다.  

어떤 부분에서 얼만큼 걸리는지 알려면 실제 수행을 해야하기 때문에 `explain analyze` 역시 실제 쿼리를 수행한다.  
따라서 어느정도는 쿼리튜닝을 한 뒤에 확인하는게 좋다.  

먼저 아래의 쿼리로 연습해보자.
~~~sql
explain analyze
select e.emp_no, avg(s.salary)
from employees e
    inner join salaries s on s.emp_no = e.emp_no
    and s.salary  > 50000
    and s.from_date  <= '1990-01-01'
    and s.to_date  > '1990-01-01'
where e.first_name = 'Matt'
group by e.hire_date
;
~~~


결과를 출력하면 아래와 같다.
`A~F`까지의 알파벳은 분석을 위해 덧붙인 것이다.

~~~sql
A: -> Table scan on <temporary>  (actual time=0.003..0.049 rows=48 loops=1)
B:     -> Aggregate using temporary table  (actual time=10.326..10.459 rows=48 loops=1)
C:         -> Nested loop inner join  (cost=535.11 rows=125) (actual time=0.458..10.170 rows=48 loops=1)
D:             -> Index lookup on e using ix_firstname (first_name='Matt')  (cost=82.34 rows=233) (actual time=0.426..1.173 rows=233 loops=1)
E:             -> Filter: ((s.salary > 50000) and (s.from_date <= DATE'1990-01-01') and (s.to_date > DATE'1990-01-01'))  (cost=0.98 rows=1) (actual time=0.029..0.035 rows=0 loops=233)
F:                -> Index lookup on s using PRIMARY (emp_no=e.emp_no)  (cost=0.98 rows=10) (actual time=0.010..0.021 rows=10 loops=233)
~~~

쿼리플랜 분석의 기본은 `indent`가 많이된 것 우선, 같다면 위에서 부터 분석이 기본이다.  
답을 알기전에 먼저 순서를 예상해보라.  

정답은 `D-F-E-C-B-A`이다.  
`F`가 먼저 나왔다면 여태 쿼리플랜을 잘못분석하고 있었다고 생각해라.  

`indent` 많이된 것 우선이라면서?  

위에서 쿼리플랜 분석의 기본은 `indent가 많이된 것 우선, 같다면 위에서 부터` 라고 했다.   
앞으로는 순서를 바꿔서 읽어보자 `indent가 같은 것은 위에서부터, indent가 많이 된 것 우선`으로 말이다. 

결과론적인 분석이지만 `F`부터 읽어보면 이상한게 있다.  
`s` 테이블을 `PRIMARY`로 읽는데 `emp_no=e.emp_no` 조건으로 읽는것이다.  
`e` 테이블을 읽지도 않았는데 어떻게 `emp_no=e.emp_no` 조건으로 읽는다는 말인가?  

`D`부터 읽어보자.

D: `e` 테이블을 `ix_firstname` 인덱스를 사용하여 `first_name='Matt'` 조건으로 조회한다.  
   총 `233`개의 데이터를 조회했으며, `233`개의 데이터를 조회하는데에 `최초` 레코드발견까지는 `0.426ms`가 걸렸고, `마지막`까지 전부 조회하는데에는 `1.173ms`가 소요되었다.
F: `D`에서 조회한 `233`개의 데이터를 기준으로 `e`의 `emp_no`컬럼과 `s`의 `PRIMARY` 첫번째 컬럼인 `emp_no`를 조인하여 조회한다.   
   `e`의 `emp_no` 하나당 `s`에서는 평균 `10`개의 데이터가 조회되는 것으로 계산되었다.  
E: 쿼리의 나머지 조건인 `s.salary > 50000`, `s.from_date <= DATE'1990-01-01'`, `s.to_date > DATE'1990-01-01'` 으로 필터링하였다.  
   `3`가지 조건으로 필터링하는데 `최초` 레코드 발견까지는 `0.029ms`가 소요되었고 최종수행시간은 평균 `0.035ms`가 소요되었다.
C: `D-F-E`의 수행결과를 조인한다.
B: `C`의 결과를 임시테이블(`temporary table`)에 저장한다.  
A: `B`의 결과로 만들어진 임시테이블을 조회하여 결과를 반환한다.