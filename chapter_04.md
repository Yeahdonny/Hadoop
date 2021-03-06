# Chapter 4 YARN

YARN
    
    - 하둡의 클러스터 자원 관리 시스템
    - 맵리듀스의 성능을 높이기 위해 하둡 2에서 처음 도입


### YARN Application

Components

1. Resource Manager (RM : 리소스 매니저)

    - 클러스터 전체 자원의 사용량 관리
    - components 
        - Scheduler
        - Application Manager
        - Resource Tracker

2. Node Manager (NM : 노드 매니저)

    - 노드 당 하나씩 존재
    - 컨테이너를 구동하고 모니터링
    - components 
        - Container
        - Application Manager

- 컨테이너

    - 클러스터의 단일 노드의 자원 (메모리, CPU)

- Application Master (AM : 애플리케이션 마스터)

    - 클러스터의 애플리케이션을 실행 · 조정하는 데몬, 하나의 테스크를 관리하는 마스터 서버

- 구동 과정

![yarn이 애플리케이션을 구동하는 방식](./pictures/yarn1.png)

    - 작업이 제출되면 애플리케이션 마스터가 생성
    - 애플리케이션 마스터가 RM에 자원을 요청
    - 실제 작업을 담당하는 컨테이너를 할당받아 작업 처리
    - 이때, 컨테이너는 작업이 요청되면 생성되고, 작업이 완료되면 종료됨

>> YARN 애플리케이션은 클라이언트로부터 요청을 받으면 먼저 리소스매니저에게 전달 된다. 그리고 그 요청은 노드매니저에게 전달이 돼서 컨테이너를 실행한다. 이 때 컨테이너는 하나의 자원(cpu, disk space 등) 덩어리이다. 컨테이너가 할당 되면 리소스 매니저에게 할당 요청을 한다 (heartbeat). 만약 해당 노드에 자원이 부족하다면 얀이 다른 노드매니저에게 요창을 해서 자원을 더 요청하게 된다.

---
- 자원 요청
    - 필요한 자원의 용량 요청 가능
    - 요청에 대한 지역성 제약 표현 가능
        - 네트워크 대역폭 효율적 활용
        - 특정 노드나 랙 또는 클러스터의 다른 곳(외부 랙)에서 컨테이너를 요청할 때 사용
        - 특정 노드 - 동일한 랙의 다른 노드 - 클러스터의 임의 노드 (맵리듀스 맵태스크 실행)
        
    - YARN : 처음에 모든 요청 or 동적으로 자원 추가 요청
        - Spark : 고정 개수의 수행자 시작
        - 맵리듀스 : 시작시 컨테이너 요청, 실패시 추가 요청 가능
        
- 애플리케이션 분류 (잡의 방식 기준)
    1. 사용자의 잡 당 하나의 애플리케이션이 실행되는 방식
        - 맵리듀스 잡
    2. <sup>[1](#footnote_1)</sup>워크플로나 사용자의 잡 세션 당 하나의 애플리케이션이 실행되는 방식
        - 스파크
        - 컨테이너 재사용 가능
        - 잡 사이 공유 데이터 캐싱 가능
    3. 서로 다른 사용자들이 공유할 수 있는 장기 실행 애플리케이션
        - 임팔라
            - 여러 임팔라 데몬이 클러스터 자원을 요청할 수 있도록 프록시 애플리케이션 제공
        - 일종의 <sup>[2](#footnote_2)</sup>코디네이션 역할 수행
        - 항상 켜져 있는 AM는 빠른 응답 가능
            >∵ 새로운 AM 구동시 필요한 오버헤드 피할 수 있음

- 애플리케이션 작성
    - 기존 애플리케이션 활용
        - 잡의 <sup>[3](#footnote_3)</sup>방향성 비순환 그래프(DAG)
           > ex) 스파크, 테즈
        - 스트리밍 처리 
            > ex) 스파크, 쌈자, 스톰
    - 프로젝트
        - 아파치 슬라이더
            > 기존의 분산 애플리케이션을 YARN 위에서 실행하도록 해줌
        
            > 여러 사용자가 동일한 애플리케이션의 서로 다른 버전 실행 가능
        
            > 애플리케이션의 노드 수 변경, 실행 제어 가능
        - 아파치 트윌
            > YARN에서 실행되는 분산 애플리케이션을 개발할 수 있는 간단한 프로그래밍 모델 추가 제공

            > 실시간 로깅, 명령 메시지 기능 제공
    - 분산 쉘
        > 클라이언트 또는 AM가 YARN 데몬과 통신하기 위해 YARN의 클라이언트 API를 어떻게 사용하는지 보여줌


### YARN과 맵리듀스 1의 차이점

> 맵리듀스 API : 사용자 측면의 클라이언트 기능, 맵리듀스 프로그램을 작성하는 방법을 결정  
맵리듀스 구현 : 맵리듀스 프로그램을 실행하는 서로 다른 방식  
>>맵리듀스 1 : 하둡 구버전의 맵리듀스 분산 구현  
맵리듀스 2 : YARN을 이용한 구현  


|MapReduce1|YARN|
|:---:|:---:|
|잡트래커|RM, AM, 타임라인 서버|
|태스크트래커|NM|
|슬롯|컨테이너|

- 맵리듀스 1
    - 잡의 실행 과정 제어 : 잡트래커, 태스크트래커
        - 잡트래커 
            1. 잡 스케줄링  
            2. 태스크 진행 모니터링  
            3. 완료된 잡에 대한 잡 이력 저장 (타임라인 서버 → 애플리케이션 이력 저장) 
                > 잡트래커의 부하를 줄이기 위해 별도의 데몬인 히스토리 서버를 통해 수행될 수 있음
        - 태스크 트래커 
            1. 태스크 실행 및 진행 상황 잡트래커에 전송

- YARN 사용시 얻는 이익
    - 확장성
        > 10,000 노드와 100,000 태스크까지 확장 가능
    - 가용성 
        > RM에 HA 제공 후 AM에 HA 제공
    - 효율성
        > 자원 할당에 대해 따로 자격이 필요하지 않음   
        필요한 만큼 자원 요청 가능
    - 멀티테넌시 (다중 사용자)
        > 버전 상관 없이 다양한 분산 애플리케이션 수용 가능  
        but, 일부 맵리듀스 or YARN 자체 업그레이드시 별도의 이중화된 클러스터 필요 


### YARN 스케줄링

    - YARN 스케줄러 역할 : 정해진 정책에 따라 애플리케이션에 자원 할당  
    - 스케줄러와 설정 정책을 사용자가 직접 선택하도록 기능 제공

- 스케줄러 
1. FIFO Scheduler
    - 공유 클러스터 환경에 적합하지 않음
    - 대형 애플리케이션 수행시 모든 자원 점유 문제
2. Capacity Scheduler
    - 기본 스케줄러
    - 트리 형태로 계층화된(분리) 전용 큐 사용
    - 큐 별로 지정된 자원 할당
    - 큐의 탄력성 : 여분의 자원 할당 (조직간 클러스터 가용량 공유 가능)
        > 다른 큐의 가용량을 잡아먹을 수 있음 ∴ 최대 가용량 설정
    - 큐의 계층구조, 가용량, 할당 자원 최대 개수, 동시 실행될 애플리케이션 개수, 큐의 접근제어 목록(ACL) etc 제어 가능

3. Fair Scheduler
    - 사용자 큐 내에 자원 균등 공유
    - 처음 애플리케이션을 제출할 때 동적으로 큐 생성
    - 큐 설정
        - 계층적인 큐 설정 가능
        - 각 큐별 서로 다른 스케줄링 정책 설정 가능
            > 1. FIFO  
            > 2. Dominant Resource Fairness (DRF : 우성 자원 공평성)
        - 최대 실행 애플리케이션의 개수, 최소·최대 자원 사용량 지정 가능
            > 최소 자원 사용량 = 자원 할당의 우선순위 (선점에 활용 가능, 클 수록 우선 할당)
    - 큐 배치
        - 맞는 규칙이 나올 때 까지 시도
        1. specified : 지정된 큐에 배치 (default)
        2. primaryGroup : 사용자의 유닉스 그룹의 이름을 가진 큐에 배치
        3. default : catch-call, 항상 dev.eng 큐에 배치
            > 모든 애플리케이션에 균등한 자원 공유 가능   
    - 선점
        - 균등 공유 기준을 위배하는 경우 컨테이너 죽이기 허용하는 기능
            > 중단된 컨테이너는 반드시 다시 수행돼야 하므로 클러스터 전체 효율 ↓
        - 타임아웃 설정 (큐 별로 설정 가능)
            1. 최소 공유 선점 타입아웃
                > 큐가 최소 보장 자원을 받지 못한 채 지정된 시간이 지날 경우 다른 컨테이너 선취
            2. 균등 공유 선점 타임아웃(default : 0.5s)
                > 큐가 균등 공유 절반 이하로 있는 시간이 지정된 시간을 초과하면 다른 컨테이너 선취

- 지연 스케줄링
    - 다른 위치에 할당하지 않고 대기함으로써 클러스터 효율성 ↑
    - NM와 RM가 주고 받는 하트비트를 통해 스케줄링 기회를 잡음 (최대 허용 횟수까지 대기한 후)
        > 하트비트 : 실행중인 컨테이너의 정보와 새로운 컨테이너를 위한 가용 자원에 대한 정보 주고 받음
    - Capacity Scheduler
        - 대기 후 제약 수준을 낮추는 기회 없음
    - Fair Scheduler
        - 기회 허용 횟수를 클러스터 크기의 비율로 정할 수 있음
        - 대기 후 동일한 랙의 다른 노드 허용

- 우성 자원 공평성 (DRF)
    - 기존에는 메모리양을 기준으로 자원 할당
    - DRF 활성화 시, 각 사용자의 우세한 자원을 기준으로 자원 할당




<br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/><br/>

---

<a name="footnote_1">1</a> : 작업의 흐름

<a name="footnote_3">3</a> : 방향 사이클이 없는(자기 자신으로 돌아오는 경로가 없는) 그래프 


[동작 관련]

https://blog.skcc.com/1884

https://box0830.tistory.com/258


=====================================================================================