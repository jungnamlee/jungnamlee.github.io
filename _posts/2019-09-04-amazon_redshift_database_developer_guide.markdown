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
WorkLoad Management(WLM)은 워크로드의 우선 순위를 관리해서 아주 작고 간단한 쿼리가 먼저 실행된 큰 쿼리에 블락되지 않도록 함. 쿼리 큐들을 관리하며 각 큐별로 동시성등 다양한 설정 가능. 디폴트는 자동이지만 수동으로도 설정 가능.

### Using Amazon Redshift with Other Services
SSH를 통하여 원격의 호스트로부터 로드 혹은 S3, DynamoDB등의 타 AWS 서비스로부터 로드, 또는 타 서비스로 언로드 가능 (서비스별로 상이)

## Proof of Concept Playbook
POC를 위한 고려 요소등을 다루는데 처음 접하는 사람에게 좋을 만한 정보가 보여서 요약해보겠음.

### Identifying the Goals of the Proof of Concept
POC의 목표가 뭐일지를 정하는게 중요하다고 한다. 스킵.

### Setting Up Your Proof of Concept
#### Designing and Setting Up Your Cluster
클러스터 셋업 시, 2 가지의 노드 타입이 있음:
- **Dense Storage**: 아주 많은 사이즈의 DW를 저렴하게 사용 가능 - HDD 사용
- **Dense Compute**: 고성능의 DW에 적합 - 빠른 CPU, 큰 용량의 RAM, SSD 사용

추가로 고려할 만한 사항:
- 클러스터 사이즈: 최소 노드 2개 필요 - 리더 노드는 추가 비용없이 포함 됨
- 클러스터를 VPC에 구축: EC2-Classic 보다 더 나은 성능 제공
- 최소 20 프로 또는 가장 큰 테이블이 요구하는 메모리의 3배 만큼의 여유 공간 확보 - 아래를 제공하는데 필요함
   - 테이블 사용 및 재작성에 사용될 공간
   - Vacuum 작업 및 테이블 재정렬에 사용될 공간
   - 쿼리 중간 결과 저장을 위한 임시 테이블

#### Converting Your Schema and Setting Up the Datasets
AWS Schema Conversion Tool (AWS SCT) 또는 the AWS Database Migration Service (AWS DMS를 이용해서 기존 스키마, 코드, 데이터 변환 가능.

### Cluster Design Considerations
5개의 속성을 고려해야 하는데 **SET DW**를 기억하면 쉽다:
- **S** - Sort Key는 다음을 고려해서 정하면 좋은데, 자세한건 다음 베스트 프랙티스 챕터내 Choose the Best Sort Key 섹션을 참고 할 것:
   - 최대 3개 까지 Sort Key로 사용할 칼럼을 선택
   - 특별성을 기준으로 오름차순으로 나열하되, 빈도 또한 고려할 것
- **E** - Encoding은 각 칼럼, 테이블 별 압축 알고리즘을 결정하는데, 자동으로도 설정 가능함
- **T** - Table 유지보수: 쿼리 통계가 최신으로 유지되면, 쿼리 옵티마이저는 더 효율적인 실행 계획을 생성 가능. 테이블 데이터에 변화가 있을시, `ANALYZE` 명령으로 통계 데이터 갱신 가능. `VACUUM` 명령으로 스캔할 블락 수를 최소화 할 수 있는데, 이 명령은 다음을 수행함:
   - 블락에서 논리적으로 삭제된 Row들을 제거하여 블락 수 감소 시킴
   - 데이터를 Sort Key 순서대로 유지해서 특정 블락들만 검색 할 수 있게 도와줌
- **D** - 테이블 Distribution, 3가지 옵션이 있음:
   - KEY: 분산할 칼럼을 지정
   - EVEN: Round-Robin 기반으로 Compute Node 지정
   - ALL: 각 Compute Node의 Database Slice에 테이블 사본 유지
   - 최적의 분산 패턴을 고르기 위해 다음을 고려할 것:
      - `Customers` 테이블을 조인 시 `customer id` 값을 자주 이용한다면, Slice별로 골고루 분산 시키기 위해 `customer id`를 분산 키로 사용하는 것을 추천
      - 테이블이 5백만 Row 정도에 차원 데이터를 포함하고 있다면, `ALL`을 사용하길 권장
      - `EVEN`은 안전한 선택이지만, 항상 모든 노드에 분산 된다는 것을 생각할 것
- **W**
   - WorkLoad Management!

### Amazon Redshift Evaluation Checklist
Skip 예정

### Benchmarking Your Amazon Redshift Evaluation
Skip 예정

## Amazon Redshfit Best Practices
### Amazon Redshift Best Practices for Designing Tables
#### Take the Tuning Table Design Tutorial
"Tutorial: Tuning Table Design" 참조

#### Choose the Best Sort Key
Sort Key가 데이터 저장시 정렬 순서를 결정하고, 쿼리 옵티마이저가 이 정렬 순서를에 쿼리 최적화에 활용. 정렬 키 선택시 다음을 고려:
- 최근 데이터를 주로 쿼리한다면 Timestamp 칼럼을 Sort Key의 선행 칼럼으로 설정
- 한 칼럼에 대해 범위 혹은 = 으로 쿼리시 해당 칼럼을 Sort Key로 설정
   - 해당 칼럼의 Min, Max 값 정보가 저장되기에 전체 블락 스캔할 필요가 없어짐
- 주로 테이블 조인 시, 조인 칼럼을 Sort Key 및 Distribution Key로 지정
   - 느린 해시 조인이 아닌, Sort 과정 없이 Sort merge 조인을 활용하게 됨

#### Choose the Best Distribution Style
쿼리 옵티마이저는 조인이나 Aggregation이 필요 할때, Row들을 컴퓨트 노드들로 분산 시킴. Distribution Style 선택의 목표는 데이터를 필요한 곳에 위치 시킴으로써 쿼리 실행시 데이터 재분배를 최소화 하는 것.

1. 팩트 테이블과 공통의 칼럼을 가진 하나의 차원 테이블과 같이 분산
팩트 테이블은 Distribution Key를 하나만 가질 수 있음. 해당 키가 아닌 다른 키에 조인하는 경우는 해당 팩트 테이블과 같이 배치되지 않게 됨.
조인 빈와 조인 될 Row들의 사이즈를 고려하여 같이 배치 시킬 하나의 차원을 정함. 차원 테이블의 Primary Key와 이와 연결 된 팩트 테이블의 Foreign Key를 DISTKEY로 설정.

2. 필터된 데이터 집합의 크기를 기준으로 가장 큰 차원을 선택
조인에 사용되는 행만 분산시켜야 하기 때문에 테이블 크기가 아닌 필터 된 데이터의 크기를 고려

3. 필터된 결과에서 Cardinality가 높은 칼럼을 선택
예를 들어, Sales 테이블을 Date 칼럼 기준으로 분산 시, 특정 시즌에만 판매가 이뤄진게 아닌 이상 데이터가 고르게 분포 될 것. 그러나 
Range 조건으로 해당 Date를 쿼리한다면 대부분의 필터 된 Row들은 제한된 Slice에 존재하게 되며 쿼리 워크로드가 편향되게 됨.

4. 일부 차원 테이블을 ALL 분산을 사용하도록 변경
만약 차원 테이블이 팩트 테이블과 같이 배치 될 수 없다면, 모든 노드에 분산 시키는 것으로 쿼리 성능을 대폭 향상 시킬 수 있음.
하지만 ALL 분산은 스토리지 용량, 로딩 시간, 유지보수 부담을 증가 시키는 점 또한 고려 할 것.

Redshift가 적절한 Distribution 스타일을 고르게 하려면 DISTSTYLE을 사용하지 말 것!

#### Use Automatic Compression
#### Define Constraints
#### Use the Smallest Possible Column Size
#### Using Date/Time Data Types for Date Columns

### Amazon Redshift Best Practices for Loading Data

### Amazon Redshift Best Practices for Designing Queries

### Working with Advisor


## Managing Database Security


## Designing Tables


## Tuning Query Performance



{% if page.comments %}
{% include disqus.html %}
{% endif %}
