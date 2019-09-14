---
layout: post
title:  "Amazon Redshift Database Developer Guide 한글 요약"

date:   2019-09-04 09:20:00 -0700
comments: true
categories: tech
---

업무용으로 필요한 챕터만 요약 중입니다.  
Internal에 관련된 챕터만 아주 대~충 요약하겠습니다. 원본 [링크](https://docs.aws.amazon.com/redshift/latest/dg)

------

## Amazon Redshift System Overview
DW용 RDBMS - 라는 말은 없지만 처음부터 알려주면 좋겠다 싶었음.

### Data Warehouse System Architecture
![](/assets/images/02-NodeRelationships.png)

#### Client applications
BI, 분석 툴등이 주 대상. Postgresql을 기반으로 만들어졌기 때문에 SQL을 사용하는 애플리케이션이라면 최소한의 수정만으로 Redshift 사용 가능.

#### Connections
JDBC, ODBC 이용

#### Clusters
Cluster = Compute nodes + 1 Leader node

#### Leader Node
Client와 Communication 담당. Query plan, compile, Compute Node로의 분배.
특정 함수 실행 및 Table(e.g. PG_%)으로의 접근은 Leader Node에서만 실행 됨.

#### Compute Nodes
컴파일된 코드 실행 후 결과를 Leader Node한테 돌려줌.

#### Node slices
Compute Node는 여러 Node slice로 구성되며 각 slice는 해당 Compute Node에 할당된 워크로드를 병럴 처리.
테이블 생성시 특정 칼럼을 Distribution Key로 설정 할 수 있는데, 이에 따라 Node slice에 row가 어떻게 분배 될지 결정 됨(자세한건 Distribution Key 참고).

#### Internal network
Leader Node와 Compute Node간에는 아주 빠른 Private 네트워크로 구성되어 있으며, Client의 직접 접근이 불가능 함.

#### Databases
Cluster는 1개 이상의 Database를 포함 할 수 있으며, User 데이터는 Compute Node에 저장 됨.
Redshift는 Postgresql 8.0.2 기반으로 만들어진 RDBMS이기에 다른 OLTP성 애플리케이션들과 호환이 가능은 하나, 고성능 분석 & 대용량 리포팅에 최적화 되었음.

### Performance
#### Massively Parallel Processing
말 그대로 여러 Compute Node에서 병렬로 실행해서 거대한 양, 복잡한 쿼리를 빠르게 처리 할 수 있게 함.
Redshift는 여러 Compute Node에 걸쳐 Row를 분산 시키는데, 이를 위해 Table 별로 적절한 Distribution Key를 잡아주는 것이 워크로드도 줄이고 노드간 데이터 이동도 감소시킴.

#### Columnar Data Storage
칼럼 형태로 데이터를 저장하는 것이 분석용 쿼리 최적화에 직결되는데, 주된 이점으로는 I/O 요청 수와 사이즈 감소가 있다 (다음 챕터에서 자세히 다룸).

#### Data Compression
적용 시 Disk I/O가 감소하여 쿼리 성능이 향상 됨. 칼럼 저장 방식이 비슷한 데이터를 연속으로 저장하기 떄문에 압축과도 연관이 있음.

#### Query Optimizer
쿼리 실행 엔진은 MPP, 칼럼 형태 저장의 특성을 이용하는 쿼리 옵티마이저를 사용함

#### Result Caching
특정 타입의 쿼리 결과는 Leader Node에 캐싱되는데, 다음 조건들이 모두 만족 할때만 캐싱한 결과를 리턴함:
- 요청한 유저에게 쿼리에서 사용 될 오브젝트들에 대한 접근 권한이 있음
- 쿼리에 쓰인 Table 또는 View가 변경 된 적 없음
- 매 실행 마다 Evaluate 되야 할 함수가 사용 되지 않음 e.g. GETDATE
- 쿼리가 Amazon Redshift Spectrum 외부 테이블을 참조하지 않음
- 쿼리 결과에 영향을 줄 만한 설정 파라미터가 바뀌지 않음
- 요청된 쿼리와 캐싱 된 쿼리가 문법적으로 일치

이외에도, 캐싱 여부를 결정하는 많은 Factor들이 있음. e.g. 너무 큰 쿼리 결과는 캐싱하지 않음

#### Compiled Code
Leader Node가 최적화와 컴파일을 마친 코드를 Cluster내 모든 노드에 배포함. 재실행시 컴파일 타임이 없어지므로 실행 시간 향상시킴. 컴파일 된 코드는 캐싱되며 같은 Cluster내 세션들과 공유 됨. Connection protocol별로 컴파일 코드가 다름. e.g. JDBC vs ODBC


### Columnar Storage
위에서도 언급했듯이 칼럼 형태 저장방식은 I/O를 줄이기 위한 중요한 팩터임.

아래 그림이 Row 형태로 데이터 블락을 저장하는 방식임:
![](/assets/images/03a-Rows-vs-Columns.png)
OLTP성 애플리케이션은 트랜잭션마다 주로 단일 혹은 적은 양의 Row를 필요로 하고 Row내 대부분의 Value를 사용하기 때문에 행 기반 저장방식은 OLTP에 최적화 되있다고 볼 수 있음.

아래는 Column 기반 저장 방식을 묘사:
![](/assets/images/03b-Rows-vs-Columns.png)
- 필드에 대해 쿼리시 I/O 감소: 예제에서 같은 타입의 데이터를 Row 기반 대비 3배 더 많이 저장하는 것을 볼 수 있다.
- 블락내 같은 필드 데이터들이 저장되기에 압축이 용이해짐
- 필요한 필드=데이터 블락만 로드하기에 메모리 효율성도 증가 - 칼럼이 매우 많은 큰 테이블의 경우 효율성은 더욱 커짐 e.g. 100 칼럼 중 5 칼럼만 필요시 칼럼 기반은 5%만 로드하면 되나, 행 기반은 전부 로드해야함

### Internal Architecture and System Operation
![](/assets/images/05-InternalComponents.png)

### Workload Management
WorkLoad Management(WLM)은 워크로드의 우선 순위를 관리해서 아주 작고 간단한 쿼리가 먼저 실행된 큰 쿼리에 블락되지 않도록 함.


### Using Amazon Redshift with Other Services


## Proof of Concept Playbook


## Amazon Redshfit Best Practices


## Managing Database Security


## Designing Tables


## Tuning Query Performance



{% if page.comments %}
{% include disqus.html %}
{% endif %}
