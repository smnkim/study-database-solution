---
layout: post
title: 3주차
---

## 조우희 : 261 ~ 322 (3.3.8절)

### 3.2.4.1. 조건 연산자별 비트맵 실행계획

![image1](https://github.com/nice21cy/study-database-solution/blob/master/images/db1.JPG)


```
비트맵인덱스 생성 절차
1. 인덱스를 생성하고자 하는 컬럼의 값들을 찾기 위해 테이블 스캔을 한 후 
2. bitmap generator에 의해 컬럼값, start rowid, end rowid , bitmap을 갖는 인덱스 엔트리를 생성한다. 
3. 2단계에서 생성된 Bitmap들을 B-tree구조에 넣기 쉽도록 key값과 start rowid 순으로 정렬한다.
4. 마지막 단계에서는 정렬된 인덱스 엔트리들을 단순히 B-tree구조로 삽입한다.
```


가) 동치(Equal) 비교 실행계획

가장 단순한 형태로 하나의 컬럼 '=' 로 비교한 경우  
'IN' 사용 시 여러개의 '=' 를 사용한 것과 동일  
비트맵 검색 처리를 'SINGLE VALUE'로 나타냄  


나) 범위(Range) 비교 실행계획  

범위를 나타내는 BETWEEN, LIKE, >, <, >=, <= 연산자 사용 시 'RANGE SCAN'  
Number 타입의 컬럼에 LIKE 사용 시   
B-Tree 인덱스 : 인덱스를 사용하지 않음  ∵ 내부적인 변형 발생  
Bitmap 인덱스 : 인덱스 'FULL SCAN' 실행계획 발생  

다) AND 조건 실행계획

각 컬럼을 자신의 단위 액세스를 수행 후 그 결과를 AND 연산 실시  
비트맵을 범위 스캔 시, 각 인덱스 처리 후 비트맵 머지 결과를 'AND' 연산 실시  
NOT EQUAL 조건과 함께 AND 조건 사용 시 'BITMAP MINUS' 연산 발생  

라) OR 조건 실행계획

AND 조건 실행계획과 동일  
부정형 조건('<>') 이 OR 연산에 사용 시 하나의 비트맵 인덱스만 사용  


마) 부등식(Not Equal) 비교 실행계획

등식 조건의 엑세스 수행 결과를 'BITMAP MINUS' 를 통하여 제거 처리  
해당 컬럼이 NULL을 포함 시 NULL 값을 제거하는 작업 추가  

![sdf](https://github.com/nice21cy/study-database-solution/blob/master/images/db2.JPG)

바) NULL 비교 실행계획

비트맵 인덱스에서는 'IS NULL' 또는 'IS NOT NULL' 연산자 처리 시 NULL도 비트맵 연산에 참여  


### 3.2.4.2 서브쿼리 실행계획

B-Tree 인덱스일 때와 유사  
여러 개의 서브 쿼리가 동시 사용 될 때, 스타변형 조인(비트맵 인덱스의 특성 사용)으로 수행되도록 처리 시 효율적 실행계획 수행  

### 3.2.4.3. B-Tree 인덱스와 연합(Combine) 실행계획

B-Tree 인덱스를 비트맵으로 전환하여 비트맵 연산 수행 가능   
"BITMAP -> ROWID" , "ROWID -> BITMAP" 으로 전환 할 수 있는 특성 이용  
비트맵 인덱스가 전혀 없을 시 "INDEX_COMBINE" 힌트 사용으로 비트맵 엑세스 가능 


## 3.2.5. 기타 특수한 목적을 처리하는 실행계획

### 3.2.5.1 순환(Recursive) 전개 실행계획
edit- connect by .. start with 구문을 사용했을때의 실행계획

```
SELECT LPAD(' ', 2*(LEVEL-1))||ename, empno, sal, mgr, SYS_CONNECT_BY_PATH(last_name, '/') "Path"  
    FROM emp  
(3) WHERE job = 'CLEAK'  
(2) CONNECT BY mgr = PRIOR empno  
(1) START WITH empno = :b1;  
```
1-1. ROOT에 해당되는 start with 에 해당하는 로우를 버퍼에 저장  
1-2. 1-1에서 찾은 로우를 하나식 ACCESS ( by userd rowid )  
2-1. prior empno 값을 상수 값으로 해서 2-2를 실행할수 있도록 한다.  
2-2. 값을 찾는다. 하나 이상 발생할수 있으므로 버퍼에 저장한다.  
3 2-1, 2-2를 계속 반복해서 값을 찾는다. where 구문은 단지 filter 역활만을 한다.  

- 서브쿼리가 있는 경우  
WHERE 구문에 있는 sub-query는 filter (체크조건) 처리가 된다.  

```
 SELECT * FROM emp  
WHERE deptno IN (SELECT deptno FROM dept WHERE loc = 'BOSTON')  
CONNECT BY mgr = PRIOR empno  
START WITH empno = 7698;  
```

- 조인쿼리를 이용하는 경우

각 테이블을 전체 스캔후, start with 조건으로 검색하여 부하가 많이 발생할수도 있다.  
인라인뷰를 이용하여 선 전개후, 그 결과를 조인하는 방식으로 사용해야 한다.  
```
 SELECT e.*, d.*  
FROM (SELECT * FROM emp CONNECT BY mgr = PRIOR empno START WITH empno = 7839) e,  
     dept d  
WHERE e.deptno = d.deptno;  
```

### 3.2.5.2. UPDATE 서브쿼리 실행계획

```
 UPDATE emp e
SET sal = (SELECT AVG(sal) * 1.2 FROM bonus b  
            WHERE b.empno = e.empno  
              AND b.pay_date BETWEEN :b1 AND :b2)  
WHERE deptno IN (SELECT deptno FROM dept WHERE loc = 'BOSTON');  
```

 서브쿼리가 수행된 결과가 'No data found'이면 NULL로 갱신될수 있다.  
  이경우는 nvl을 이용해 비교할 값도 없어 null로 갱신된다.  


### 3.2.5.3. 특이한 형태의 실행계획

-서브쿼리 팩토리(Factoring)  

WITH 절을 사용하여 생성한 복잡한 쿼리 문을 임시 테이블이 실제로 저장을 해 두었다가 사용하는 기능  

```
 WITH total_sal AS  
     (SELECT d.deptno, d.loc, e.job, sum(e.sal) tot_sal  
        FROM emp e, dept d  
       WHERE e.deptno = d.deptno  
         AND e.hiredate > :b1  
       GROUP BY d.deptno, d.loc, e.job)  
SELECT e.empno, e.ename, e.sal, e.sal/t.total_sal sal_percent  
FROM emp e, total_sal t  
WHERE e.deptno = t.deptno  
AND e.sal > (SELECT max(total_sal) FROM total_sal WHERE job = 'CLERK');  
```


* 특이한 DELETE 문 서브쿼리

```
 DELETE FROM (SELECT * FROM emp  
              WHERE job = 'CLERK'  
                AND comm > 10000 AND deptno IN (SELECT deptno FROM dept WHERE loc = 'BOSTON')  
            );  
```
            
* 다중 테이블 입력(Multi-table Insert)
```
 INSERT ALL  
  INTO gplanappo.plan_minor (MINOR_CD,major_cd,org_id,created_by,creation_date,last_updated_by ,last_update_date) VALUES (num_v+1,'s',10,1,sysdate,-1,sysdate)  
  INTO gplanappo.plan_minor (MINOR_CD,major_cd,org_id,created_by,creation_date,last_updated_by ,last_update_date) VALUES (num_v,'s',10,1,sysdate,-1,sysdate)  
SELECT 1000000 AS num_v FROM dual;  
```


- Having 서브쿼리
```
 SELECT department_id, manager_id  
  FROM employees  
 GROUP BY department_id, manager_id  
 HAVING (department_id, manager_id) IN (SELECT e.deptno, e.mgr FROM emp e, dept d  
                                         WHERE e.deptno = d.deptno  
                                           AND d.loc = 'BOSTON');  
```
- cube, grouping sets, rollup 처리  

```
 SELECT co.country_region, co.country_subregion, SUM(s.amount_sold) "Revenus", GROUP_ID() g
FROM sales s, customers c, countries co
WHERE s.cust_id = c.cust_id AND c.country_id = co.country_id AND s.time_id = :b1
AND co.country_region IN ('Americas', 'Europe')
GROUP BY ROLLUP (co.country_region, co.country_subregion);
```


- merge 문

```
 MERGE INTO bonuses D
USING (
       SELECT employee_id, salary, department_id
         FROM employees
        WHERE department_id = 80
      ) s
ON (d.employee_id = s.employee_id)
WHEN MATCHED THEN
  UPDATE SET d.bonus = d.bonus + s.salary * 0.1;
  DELETE where (s.slary > 8000)
WHEN NOT MATCHED THEN
  INSERT (d.employee_id, d.bonus) VALUES (s.employee_id, s.salary*0.1);
```

## 3.3. 실행계획의 제어


 잘못을 바로 잡아 주는 용도보다 옵티마이져가 가지고 있지 못하는 정보를 우리가 더 많이 알고 있을 때나 특별한 목적을 관철하고자 할때 사용
 불필요한 힌트는 액세스 경로에 결정적 악영향을 미칠수도 있음

### 3.3.2. 최적화 목표(Goal) 제어 힌트

옵티마이져 모드를 바꾸어서 그 결과를 살펴보는 목적

ALL_ROWS ; Cost-based로 쿼리 전체를 최적화 할경우의 최저비용(Best throughput)의 실행계획 수립.
                         Full table scan을 선호하는 경향이 있음.
 
CHOOSE ; 통계정보 유무에 따라 규칙기준이나 비용기준을 적용. 통계정보가 충분한 경우 Cost-based 선택.

 
FIRST_ROWS ; Cost-based로 최적 응답시간(Best response time)을 목표로 최저 비용의 실행계획을 수립하도록 유도.
                           Index Scan을 선호하는 경향이 있음.

 
RULE ; 규칙 기준 옵티마이져를 이용한 최적화를 요구

 
### 3.3.3. 조인 순서 조정을 위한 힌트

다수의 테이블을 조인하는 경우에 조인 순서에 혼선이 있을 때 적용

ORDERED ; FROM 절에 기술된 테이블 순서대로 조인을 수행하도록 유도
          조인 방법을 유도하기 위한 USE_NL, USE_MERGE 등의 힌트와 함계 사용

LEADING ; FROM 절에 기술한 테이블의 순서와 상관없이 조인 순서를 제어


### 3.3.4. 조인 방법 선택용 힌트

USE_NL ; Nested Loops 방식을 사용하여 조인을 수행

NO_USE_NL ; Nested Loops 방식을 제외한 다른 방식의 조인을 사용하도록 제시

USE_NL_WITH_INDEX ; USE_NL과 INDEX 힌트를 하나로 통합한것 (NL 조인에서 외측루프의 주관 인덱스를 지정)
    
USE_HASH ; 해쉬 조인 방식으로 조인이 수행되도록 유도

NO_USE_HASH ; 해쉬 조인을 제외한 다른 방식의 조인을 고려토록 유도

USE_MERGE ; Sort Merge 방식으로 조인을 수행하도록 유도 , 필요시 'ordered' 힌트와 같이 사용

NO_USE_MERGE; Sort Merge 방식을 제외한 다른 방식의 조인 고려토록 유도

### 3.3.5. 병렬처리 관련 힌트

시스템 자원을 최대한 사용하여 결과를 얻는 절대 시간을 줄이겠다는 목적

PARALLEL ; 대량의 DATA에 대한 TABLE을 액세스 할때와 DML을 처리할때 SQL의 병렬처리를 지시하는 힌트
                  PARALLEL THREADS를 나타내는 숫자(PARALLEL DEGREE)와 함께사용, Parallel Degree를 지정하지 않으면 
                  PARALLEL_THREADS_PER_CPU 파라메터에 정의된 값을 이용
                 TABLE 정의시 PARALLEL 을 지정하였다면 힌트없이 적용 
                 DML 문에 적용시는 ALTER SESSION ENABLE PARALLEL DML' 을 지정해야함

NOPARALLEL ; 해당 테이블의 PARALLEL  파라메터를 무시하고 병렬처리를 하지 않는 실행계획을 수립

PQ_DISTRIBUTE ; 슬레이브 프로세스 ( 생산자(Producer)와 소비자(Consumer)프로세스) 사이에서 조인할 테이블 로우를 할당작업(Distribution)을 하는 방법을 정의하는 힌트

PARALLEL_INDEX ; 파티션 인덱스에 대한 인덱스 범위 스캔을 병렬로 수행하기 위한 병렬도를 지정

NOPARALLEL_INDEX ; parallel 파라메터를 무시하여 병렬 인덱스 범위 스캔을 하지 않음
                                버젼에 따라 NO_PARALLEL_INDEX


### 3.3.6. 액세스 수단 선택을 위한 힌트


FULL ; 전체 테이블 스캔 방식으로 유도

HASH ;  해쉬 스캔 방식으로 액세스 하도록 유도하는 힌트

CLUSTER ; 클러스터 인덱스를 통해 스캔하도록 유도

INDEX ; 인덱스 범위 스캔을 유도 (뷰의 경우 뷰쿼리 내부의 테이블 인덱스 지정가능)

NO_INDEX ; 지정한 인덱스를 제외하고, 다른 액세스 방법을 고려하도록 유도
                   INDEX,INDEX_ASC,INDEX_DESC,INDEX_COMBINE,INDEX_FFS등을 함께사용하면 두 힌트 모두 무시

INDEX_ASC ; 지정한 인덱스를 인덱스 컬럼 값의 오름차순으로 범위 스캔하도록 유도

INDEX_DESC ; 지정한 이덱스를 인덱스 컬럼값의 내림차순으로 범위 스캔하도록 유도

INDEX_COMBINE ; 2개 이상의 인덱스를  비트맵 인덱스로 변경/결합하여 테이블을 액세스 하는 방식으로 유도하는 힌트

INDEX_FFS ; 인덱스 전체 범위를 스캔하는 방식으로 유도하는 힌트, 다중 블록을 스캔

NO_INDEX_FFS ; (Fast Full Scan) 고속 전체 인덱스 스캔 방식을 제외한 다른 액세스 방법을 사용하도록 유도

INDEX_JOIN ; 두개 이상의 인덱스들만으로 조인을 수행하도록 유도 ( 인덱스가 필요로 하는 모든 컬럼의 값을 가져야함)

INDEX_SS ; 인덱스 스킵 스캔 방식으로 인덱스를 액세스 하도록 유도

NO_INDEX_SS ; (Skip Scan) 지정한 테이블의 인덱스에 대한 스킵스캔을 제외한 다른 액세스 방법을 사용하도록 유도

INDEX_SS_ASC ; 인덱스 스킵 스캔 방식에서 오름차순으로 읽도록 유도

INDEX_SS_DESC ; 인덱스 스캡 스캔 방식에서 내림차순으로 읽도록 유도

### 3.3.7. 쿼리형태 변형(Query Transformation)을 위한 힌트


USE_CONCAT ; OR 이나 IN 연산자를 별도의 실행 단위로 분리하여 실행계획을 수립하고 연결하는 통합실행계획을 수립하도록 유도

ex) SELECT /*+USE_CONCAT */
    FROM emp
    WHERE job ='CLERK' OR deptno=10;

NO_EXPAND ; OR 이나 IN을 연결 실행계획으로 처리되지 않도록 사용 (USE_CONCAT의 반대)

ex) SELECT /*+ NO_EXPAND */ ..
    FROM CUSTOMER
    WHERE CUST_TYPE IN ('A','B');

REWRITE ; M-VIEW(실체뷰) 를 사용할때 뷰에 사용된 테이블을 직접액세스하거나 M-VIEW를 액세스하는 것중 유리한것을 석택하도록 쿼리 재작성작업을 함

NOREWRITE ; QUERY_REWRITE_ENABLED 파라메터가 TRUE로 정어되어있어도 이를 무시 
                     M-view가 있어도 원 테이블로 직접 계산을 유도함으로 최신값으로 결과로 출력 할수있음

MERGE ; 뷰쿼리 병합이 가능함에도 불고하고 뷰쿼리 병합이 일어나지 않을 경우


NO_MERGE ; 뷰쿼리 병합이 일어나지 않도록 요구 하는 힌트


STAR_TRANSFORMATION ; 소량의 데이터를 가진 여러개의 디멘전 테이블과 팩트 테이블의 개별 비트맵 인덱스를 이용하여 처리 범위를 줄이는 조인방식

```
 SELECT /*+ STAR_TRANSFORMATION */
                d.dept_name, c.cust_city, p.product_name, SUM(s.amount) sales_amount
    FROM SALES s, PRODUCTS t, CUSTOMERS c, DEPT d
    WHERE s.product_cd = t.product_cd
        AND s.cust_id = c.cust_id
        AND s.sales_dept = d.dept_no
        AND c.cust_grade between '10' and '15'
        AND d.location = 'SEOUL'
        AND p.product_name IN ('PA001','DR210')
    GROUP BY d.dept_name, c.cust_city, p.product_name;
```

NO_STAR_TRANSFORMATION ; 스타변형 조인을 하지 않도록 유도

FACT ; 스타 변형 조인에서 팩트 테이블을 지정하기 위하 사용
NO_FACT ; 지정한 테이블을 팩트 테이블로 인정하지 말아 달라고 요구

UNNEST ; 서브 쿼리와 메인쿼리를 합쳐 조인 형태로 변형하도록 하는 실행계획을 생성하도록 유도하는 힌트

NO_UNNEST ; UNNESTING 을 하지 않도록 유도

### 3.3.8. 기타 힌트


APPEND ; INSERT문에서 사용, SGA를 거치지 않고 'DIRECT-PATH'방식으로 직접 저장공간으로 입력 (최고 수위점 다음위치에 저장)

NOAPPEND ; INSERT시 'CONVENTIONAL-PATH'방식으로 수행 (직렬모드)

CACHE ; FULL 스캔 방식으로 읽혀진 블록을 LRU 리스트의 최근 사용위치에 머물도록 하여 계속 메모리에 머물수 있도록 처리

NOCACHE ; FULL 스캔방식으로 읽혀진 블록을 LRU 리스틔 끝에 위치하도록 유도(메모리에서우선제거), 옵티마이져가 블록을 관리하는 일반적방법

CARDINALITY ; 옵티마이져에게 해당쿼리나 일부구성에 대한 카디널리티 에상값을 제시하여 실행계획수립에 참조토록 함 
                       테이블을 지정하지 않으면 전체 쿼리를 수행한 결과로 얻어진 총건수로 간주

CURSOR_SHARING_EXACT ; CURSOR_SHARING 초기화 파라메터가 'EXACT' 일경우와 동일한 결과 , 리터럴 값을 바인드 변수로 변경하지 않음

DRIVING_SITE ; 원격 테이블과 조인을 수행할 사이트를 지정하여 분산 쿼리를 최적화 하는데 적용

DYNAMIC_SAMPLING ; 통계정보를 가지고 있지 않거나 ,에러등의 문제로 사용할수 없게되거나, 너무 오래되어 신뢰할수 없을 때 적용
                            통계정보가 있는 하나의 테이블에 사용할경우 무시됨

PUSH_PRED ; 뷰나 인라인뷰의 외부에 있는 조인 조건을 뷰 쿼리 내로 삽입하는 힌트

NO_PUSH_PRED ; 뷰나 인라인 뷰의 외부에 있는 조인 조건을 뷰쿼리 내료 삽입하지 않도록 하는 힌트

PUSH_SUBQ ; 머지 되지않는 서브 쿼리를 최대한 먼저 수행할 수 있도록 실행계획을 수립하기를 요구 

QB_NAME ; 쿼리 블록에 이름을 부여하여 외부의 다른 힌트에서 참조할수 있도록 함

REWRITE_ON_ERROR ; 적합한 M-VIEW가 존재하지 않아 쿼리 재생성을 할수 없을 경우 
                                      ORA-30393 에러를 유발하여 쿼리 수행을 중단




## 김새미나 : 323~ 386 (4.1.6절)
범위: 323~ 386 (4.1.6절)



## 제 4장 인덱스 수립 전략

### 4.1. 인덱스의 선정 기준
최소의 인덱스로 최대의 엑세스 형태를 모두 만족할 수 있도록 하는 전략이 필요
`이 단원을 한 줄로 요약하자면, 어떤 컬럼들로 결합되었는가? 어떤 순서로 결합되었는가?`

##### 4.1.1. 테이블 형태별 적용 기준
- 적은 데이터를 가진 소형 테이블
DB_FILE_MULTIBLOCK_READ_COUNT에 지정된 숫자 이하의 불록을 가진 테이블 (한번에 I/O에서 액세스되는 블록 이내의 크기)의 경우 인덱스 유무가 성능에 거의 미치지 않는다.
하지만 다른 테이브를과 다양한 방법으로 연결되는 경우, 소형 테이블도 인덱스 구성이 바람직하다.
또한, 아무리 소형테이블이라도 기본키는 반드시 인덱스를 가지도록 한다. 기본키는 조인의 실행계획이나 각종 무결성 정의에 영향을 미친다.


- 주로 참조되는 역학을 하는 중대형 테이블
데이터는 많고 많은 랜덤 엑세스가 발생하지만 주기적으로 대량의 데이터가 입력되는 경우가 없는 테이블
즉, 검색 조건의 형태가 뚜렷하고 데이터 증감이 별로 없으며 검색 위주의 액세스가 발생하는 경우
이런 테이블의 인덱스는 설사 인덱스 개수가 좀 많아진다 하더라도 역할에 좀더 잘 맞는 구성으로 과감하게 결정 가능
ex) 고객테이블


- 업무의 구체적인 행위를 관리하는 중대형 테이블
시간이 지남에 따라 데이터가 지속적으로 증가하는 테이블
ex) 매출정보테이블
`인덱스 전략 수집을 위한 절차를 충실히 지켜야 한다.`
    1. 해당 테이블을 액세스하고 있는 모든 유형을 수집한다.
    2. 이 유형들을 가장 이상적으로 만족시킬 수 있는 인덱스 조합을 찾아낸다.
    3. 기존에 사용된 액세스 형태뿐만 아니라 앞으로 예상되는 형태까지도 감안한 인덱스를 구성한다.
    \* 새로운 액세스 유형이 나타났다고 해서 개발자가 해당 쿼리의 최적화만을 위해서 함부로 인덱스를 추가하도록 해서는 안된다.
    


- 저장용 대형 테이블
ex) 로그성테이블


##### 4.1.2. 분포도와 손익분기점
WHERE 절에는 처리범위를 주관하는 것과 단순히 체크 기능만을 담당하는 조건이 있다.
최선의 방법은 가장 처리범위가 적은 것을 처리주관으로 하는 것. 결국 어떻게 인덱스를 구성해야 가장 최소의 범위를 처리할 수 있는가에 달려있다.
인덱스 컬럼의 분포도가 10~15%를 넘지 않아야 인덱스로서의 가치가 있으며 해당 수치가 `손익분기점`을 의미한다.
인덱스 컬럼의 분포도가 10~15%가 기준이라는 것은 그 이상인 경우는 차라리 전체 테이블을 스캔하는 것이 유리하다.


##### 4.1.3. 인덱스 머지와 결합 인덱스 비교 (339p)
인덱스 머지(Index Merge) : 여러 인덱스가 협력하여 같이 액세스를 주관. 머지할 대상이 서로 비슷한 분포도를 가지고 있을 때 유리
결합 인덱스(Concatenated Index) : 여러 개의 컬럼을 모아 하나의 인덱스로 생성시키는 것. 그외에는 주로 결합 인덱스를 사용

##### 4.1.4. 결합 인덱스의 특징
결합 인덱스의 첫 번째 컬럼이 WHERE절에 없다면 일반적으로 인덱스는 사용되지 않는다.
결합 인덱스의 최대의 단점은 컬럼 중 일부만 조건을 받거나 '='이 아닌 연산자가 만이 있으면 급격하게 처리범위가 증가하여 효율이 떨어진다.
이것은 곧 `어떤 컬럼들로 결합되었는가? 어떤 순서로 결합되었는가?`에 대한 매우 치밀한 전략이 필요하다는 것을 의미한다.

- 분포도와 결합순서의 상관관계
분포도가 넓은 컬럼이 앞에 있거나 분포도가 좁은 컬림이 앞에 있거나 모두 '='(Equal)로 사용한 경우에는 실제 처리한 일량의 차이가 없다.


- 이퀄(=)이 결합순서에 미치는 영향
인덱스의 첫번째 컬럼이 '='로 사용되지 않으면 뒤에 있는 컬럼이 비록 '='로 사용되었더라도 처리범위는 줄지 않는다. 결합 인덱스의 컬럼 순서를 결정하는 매우 중요한 판단요소


- IN연산자를 이용한 징검다리 효과
BETWEEN VS IN
between, like는 선분의 개념이지만 IN은 점의 개념이기 때문에 IN으로 되도록 사용 권장


- 처리범위에 직접적인 영향을 주지 못하는 컬럼의 추가 기준 (350p)
결합인덱스가 COL1+COL2+COL3+COL4 로 되어있다고 가정
COL1은 =, COL2은 LIKE, COL3은 >, COL4는 BETWEEN으로 사용되었다면 COL1과 COL2까지는 처리범위를 줄이는데 직접적인 영향을 준다.
COL3, COL4는 앞에 위치한 컬럼이 '='이 아니므로 직접적인 영향을 주지 못하고 체크기능만 담당한다. 그럼에도 불구하고 인덱스 뒤에 추가하는 이유는?
인덱스 엑세스는 스캔 방식이고 테이블 엑세스는 랜덤 방식이기 떄문에 COL3, COL4가 인덱스내에 정렬되어 있기 떄문에 성공한 로우를 바로 추출 가능하다.




##### 4.1.5. 결합 인덱스의 컬럼순서 결정 기준
1단계 : 항상 사용하는가?
2단계 : 항상 '='로 사용하는가?
3단계 : 어느 것이 더 좋은 분포도를 가지는가?
4단계 : 자주 정렬되는 순서는 무엇인가?
5단계 : 부가적으로 추가시킬 컬럼은 어떤 것으로 할 것인가?

##### 4.1.6. 인덱스 선정 절차
인덱스를 설계하는 시점은 크게 두 가지로 나눌 수 있다.
`애플리케이션을 개발하는 단계`와 `시스템이 운영되고 있는 상태에서 시스템 최적화를 위해 인덱스를 재구성`하는 경우가 있다.

- 테이블의 액세스 형태를 최대한으로 수집
    - `개발단계의 액세스 형태 수집` 
    100%는 힘들지만 70~80% 정도의 인덱스 생성 필요
        1) 반복 수행되는 액세스 형태를 찾음
반복 수행되는 액세스는 자신의 수행속도에 반복 횟수를 곱한 만큼 부하를 가져오므로 수행속도에 미치는 영향이 아주 큼 → 기본키, 외부키, 서브쿼리의 컬럼, 루프 내 사용되는 컬럼
        2) 분포도가 아주 양호한 컬럼들을 발췌하여 액세스 유형을 조사
테이블에는 아주 양호한 분포도를 가진 컬럼이 있기 마련 → 단, 분포도 뿐만이 아니라 결합도도 체크
        3) 자주 넓은 범위의 조건이 부여되는 경우를 찾음
인덱스는 넓은 범위를 처리하게 될때 많은 부담을 주게 됨 → 처리범위의 최대 크기와 평균 예상범위, 자주 사용되는 정렬의 순서, 처리 유형(Group by, Order by) 등 활용
        4) 조건에 자주 사용되느 주요 컬럼들을 추출하여 액세스 유형을 조사
빈번하게 사용 된다는 것은 시스템 전반에 미치는 영향이 그 만큼 크기 때문에 아주 중요 → 매출 테이블의 '판매일자', '판매부서', '고객번호', '상태', '상품' 등
        5) 자주 결합되어 사용되는 컬럼들의 조합형태 및 정렬순서 조사
업무적인 측면에서 각 컬럼들간의 상호관계를 잘 파악해 보면 특정 컬럼들끼리 자주 조합하여 사용되는 결합형태를 찾을 수 있음 → 모든 컬럼들간 결합유형을 모두 취할 수 없음, 가장 중심되는 컬럼과 그와 연관성이 가장 큰 컬럼만으로 결합인덱스 생성 또는 클러스터링을 함으로 효율 ↑
        6) 역순으로 정렬하여 추출되는 경우를 찾음
일반적으로 최종 사용자는 조건의 범위는 벏게 부여하면서 결과는 가장 최근에 발생된 데이터부터 보기를 원하는 경우가 많음 → 힌트 사용으로 인덱스 역순 액세스
        7) 통계자료 추출을 위한 액세스 유형 조사
통계자료를 추출하는 경우는 대게 범위가 넓음 → 클러스터링이나 잘 조합된 결합 인덱스 이용
    - `운영단계의 액세스 형태 수집`
        1) 애플리케이션 소스코드에서 SQL을 추출하여 분석용 테이블에 보관
        2) SQL-Trace 파일을 파싱하여 SQL 문장뿐만 아니라 실행계획, 현재 적용되고 있는 인덱스, 실행횟수, 처리범위 등 매우 상세한 정보 획득 가능

    - `수집된 SQL을 테이블 별로 출력하여 액세스 형태 기록`
        
- 인덱스 대상 컬럼의 선정 및 분포도 조사
액세스 유형에 자주 등장하는 컬럼
인덱스의 앞부분에 지정해야 할 컬럼
기타 수행 속도에 영향을 미칠 것으로 예상되는 컬럼
- 특수한 액세스 형태를 최대한으로 수집
반복 액세스 되는 형태(Critical Access Path)를 찾음
반복 수행 액세스는 수행속도에 대하여 반복 수의 배수만큼 증가하기에 0.02초를 0.01초를 줄이는 것이 10초를 1초로 줄이는 것보다 효과적일 수 있음
- 클러스터링 검토
인덱스의 단점
: 넓은 범위 데이터를 인덱스를 경유해 찾을 경우 대량의 랜덤 액세스 발생으로 수행속도 급감
클러스터링으로 넓은 범위의 데이터 효과적 액세스
- 결합 인덱스 구성 및 순서의 결정
각 컬럼간 상호관계를 파악하고 결합 인덱스 구성 및 순서를 결정하여 액세스 수행 속도가 모두가 만족 되도록 함
- 시험생성 및 테스트
인덱스 전략이 새롭게 수립되었으면 운영 중인 시스템에 바로 적용시키지 말고 테스트 과정을 거치도록 한다.
- 수정이 필요한 애플리케이션 조사 및 수정
- 일괄적용
잘못된 SQL의 수정과 인덱스 변경을 동시에 일괄적으로 적용
SQL의 잘못된 점은 수정하기 쉬우나 인덱스는 쉽지 않음
인덱스 수정 시, SQL 수정 발생
