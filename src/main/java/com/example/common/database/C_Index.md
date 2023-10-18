#### InnoDB Index 구조
~~~sql
CREATE TABLE `tb_user` (
  `id` int NOT NULL,
  `emp_no` int DEFAULT NULL,
  `first_name` varchar(10) COLLATE utf8mb4_general_ci DEFAULT NULL,
  `last_name` varchar(20) COLLATE utf8mb4_general_ci DEFAULT NULL,
  `hire_date` date DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_first_name` (`first_name`)
);
~~~
위의 테이블을 예시로 인덱스를 알아보자.  
먼저 인덱스를 하나 생성한다.
~~~sql
ALTER TABLE tb_user ADD UNIQUE INDEX uidx_emp_no_last_name(emp_no, last_name);
~~~

우리는 [Record Lock](https://github.com/outgrow0905/study-common/blob/main/src/main/java/com/example/common/database/A_InnoDB-Locking.md#record-locks)에 대해서 알아본 적이 있다.  
이때에 `update`에 사용된 `인덱스`와 `PK` 둘 다에 락이 걸렸던 것을 본 적이 있다.  
그대로 위의 인덱스를 가지고 다시한번 복습해보자.  

~~~sql
select * from tb_user where emp_no  >= 10081  for update;
~~~
![index1](img/index1.png)

쿼리에 사용된 `uidx_emp_no_last_name` 인덱스에 `X lock`이 걸리는 것은 이해가 된다.  
`PK`에는 왜 걸리는 것일까?   
이는 `InnoDB`의 인덱스 구조와 관련이 있다.  
이 말은 스토리지 엔진이 `InnoDB`가 아니면 위와 같이 `PK`에 락이 안잡힐수도 있다는 의미이기도 하다.  
그렇다면 `InnoDB`의 인덱스는 어떻게 생겼는지 알아보자.

- B Tree
  ![index3](img/index3.png)
- InnoDB 인덱스
  ![index2](img/index2.png)

첫번째 그림은 일반적인 `B Tree`의 구성이다.  
두번째 그림은 `InnoDB`의 인덱스 구조이다.  
`InnoDB`는 `B Tree`구조이다. 그런데 자세히 보면 첫번째 그림 구조와 두번째 그림구조는 차이가 있다.  

첫번째 그림과 두번째 그림을 비교하며 `InnoDB` 인덱스의 세 가지 특징들을 읽어보자.  

`B Tree` 사진에서는 `1`부터 `34`까지의 모든 데이터가 각 노드에 고르게 분포되어있다.  
`인덱스` 사진에서는 `리프노드`에 모든 데이터가 포함되어있다.  
`인덱스`의 `리프노드`에 전체 데이터가 있는것은 중요한 특징중 하나이다.    

`인덱스` 사진에서 `리프노드`들 간에 `작은 화살표`가 있는 것이 보일 것이다.  
이는 `인덱스`의 `리프노드`들이 서로 연결되어있는 `LinkedList`의 형식이라는 것을 의미한다.  
예를 들어 `페이지(4)`의 리프노드에서 `페이지(5)`로 조회해야한다면 페이지를 옮겨갈때마다   
다시 루트노드부터 시작해야할 필요 없이 다음 페이지의 주소를 알고 있고 바로 옮겨갈 수 있다는 것이다.  

마지막으로 리프노드에서 `프라이머리 키`부분을 살펴보자.  
이 부분이 `InnoDB 인덱스`의 특징 중 하나이다.  
모든 데이터를 가지고 있는 리프노드에서 다시 `PK`를 참조하고 있는 것이다.  
이러한 특징으로 `보조 인덱스`를 이용한 쿼리는 결국에 `PK`를 조회하게 되는 것이다.  

이제 맨 처음 질문에서의 `보조 인덱스`로 조회 시에 `보조 인덱스`와 `PK` 전부에 `X lock`이 걸린 이유를 알게 되었다.

