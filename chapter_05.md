
# Chapter 5 하둡 I/O

Primitive  
    
    - 데이터 무결성 · 압축 (일반적인 기술)  
    - 직렬화 프레임워크나 디스크 기반 데이터 구조와 같은 분산 시스템을 개발하기 위한 구성요소 제공 (하둡 도구, API)  
    - 멀티테라바이트의 데이터셋 처리시 고려할 가치 有

### 데이터 무결성

- 손상된 데이터 검출 방법

    1. 체크섬 계산
        - 데이터가 시스템에 처음 유입되었을 때, 신뢰할 수 없는 통신 채널로 데이터가 전송되었을 때
        - 단순 에러 검출만 수행
        - 체크섬이 손상될 가능성 ↓
    2. CRC-32
        - 32비트 순환 중복 검사
        - 하둡의 ChecksumFileSystem에서 체크섬 계산을 위해 사용
        - 변형 : CRC-32C (HDFS에서 사용)

</br></br>
> 참고) 'A → B' : A가 B에게 데이터 송신, B가 A의 데이터를 읽음 

- HDFS의 데이터 무결성
    - 데이터를 쓰는 과정에서 체크섬 계산, 데이터를 읽는 과정에서 체크섬 검증
    - 클라이언트 → 데이터노드 / 데이터 노드 → 데이터노드(복제)
        - 데이터 노드 파이프라인의 마지막 노드가 해당 데이터의 체크섬 검증  
            
        - 에러 검출시, IOException, 애플리케이션 특성에 맞게 처리 (ex. 재연산 시도)
    - 데이터노드 → 클라이언트
        - 데이터노드에 저장된 체크섬과 수신된 데이터로부터 계산된 체크섬 검증 

        - 각 데이터노드는 체크섬 검증 로그를 저장 → 오류 디스크 검출에 유용

    - 데이터노드 내부
        - DataBlockScanner 백그라운 스레드 수행 : 저장된 모든 블록 주기적으로 검증 
            > <sup>[1](#footnote_1)</sup>'bit rot'에 의한 데이터 손실 방지

    - 블록 조회 (블록 → 클라이언트)
        - 정상 복제본 중 하나를 복사하여 손상된 블록 치료
        - 에러 검출시  
            1. 훼손된 블록과 데이터노드에 대한 정보를 네임노드에 보고, ChecksumException 발생  
            2. 네임노드가 블록 복제본이 손상되었다고 표시하고 다른 클라이언트에 제공하거나 또 다른 데이터노드로 복사하지 못하도록 방지
            4. 네임노드는 해당 블록을 다른 데이터노드에 복제되도록 스케줄링해서 그 블록의 복제 계수를 원래 수준으로 복구
            - 손상된 복제본은 삭제
    - 파일 조회 
        - 체크섬 검증 비활성화 가능
            > 손상된 파일을 조사할 때 이 파일로 무엇을 할 지 결정할 수 있어 유용 (ex. 파일 삭제 전 북구 여부 파악)
    - 파일의 체크섬 확인
        - 'Hadoop fs -checksum' : HDFS 안의 두  파일이 동일한 내용인지 확인할 때 유용 (ex. distcp)
        

- LocalFileSystem
    - 체크섬은 성능에 미치는 영향 미비
    - 클라이언트 측 체크섬 수행
        - 파일을 쓸 때 
            > 파일 시스템 클라이언트는 파일과 같은 위치의 디렉터리에 해당 파일의 각 청크별 체크섬이 담긴 .filename.crc라는 숨겨진 파일을 내부적으로 생성  
        
        - 파일을 읽을 때 
            > 체크섬이 검증되고 에러 검출시 LocalFileSystem이 CkecksumException 발생
    - RawLocalFileSystem 으로 LocalFileSystem의 체크섬 비활성화 가능 
        > if (기존 파일시스템이 자체적으로 체크섬 지원)

- ChecksumFileSystem
    - FileSystem의 wrapper로 구현되어 다른 체크섬이 없는 파일시스템에 체크섬 기능 쉽게 추가 가능
    - getRawFilesystem() : 내부의 파일시스템 얻기 (Raw FileSystem)
    - getChecksumFile() : 특정 파일에 대한 체크섬 경로 얻기
    - reportChecksumFailure() : 에러 검출시 호출
        > default : 아무 동작 x

        > LocalFileSystem : 문제가 되는 파일과 체크섬을 같은 디바이스의 별도 디렉터리로 이동 (bad_files) 
    - LocalFileSystem 동작할 때 ChecksumFileSystem 사용

### 압축

    - 파일 저장 공간 ↓
    - 데이터 전송 고속화

|압축포맷|도구|알고리즘|파일 확장명|분할 가능|하둡 압축 코덱|자바 구현체|원시 구현체|
|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
|DEFLATE|N/A|DEFLATE|.deflate|No|org.apache.hadoop.compress.DefaultCodec|Yes|Yes|
|gzip|gzip|DEFLATE|.gz|No|org.apache.hadoop.compress.GzipCodec|Yes|Yes|
|bzip2|bzip2|bzip2|.bz2|Yes|org.apache.hadoop.compress.BZip2Codec|Yes|Yes|
|LZO|lzop|LZO|.lzo|No|com.hadoop.compression.lzo.LzopCodec|No|Yes|
|LZ4|N/A|LZ4|.lz4|No|org.apache.hadoop.compress.Lz4Codec|No|Yes|
|Snappy|N/A|Snappy|.snappy|No|org.apache.hadoop.compress.SnappyCodec|No|Yes|

    * DEFLATE 표준 구현이 zlip인 압축 알고리즘
    * LZO파일의 경우 전처리 과정에서 색인을 생섷앴다면 분할 가능

- 공간과 시간은 트레이드오프 관계
    > 압축과 해제가 빨라질수록 공간이 늘어남  
     옵션 ( -1 : 스피드 최적화,  -9 : 공간 최적화)

- gzip
    - 일반적인 목적의 압축 도구
    - 공간/시간 트레이드오프의 중앙에 위치
- bzip2
    - gzip보다 효율적으로 압축하지만 더 느림
    - 압축 해제 속도(빠름) > 압축 속도
        - 다른 포맷에 비해 더 느림
- LZO, LZ4, Snappy
    - 속도에 최적화
    - gzip보다 빠르지만 압축 효율↓
    - 압축 해제 속도 Snappy, LZ4 (빠름) > LZO
- 분할 가능
    - 압축 포맷이 분할을 지원하는지 알려줌
    - ex) 스트림의 특정 지점까지 탐색한 후 이후의 일부 지점으로부터 읽기를 시작할 수 있는지 알려줌
    - 맵리듀스에 적합

</br>

- 코덱
    - 압축-해제 알고리즘을 구현한 것
    - CompressionCodec 인터페이스로 구현
    - LZO 라이브러리는 별도로 내려받아야 함
    - CompressionCodec을 통한 압축 및 해제 스트림
        - createOutputStream(OutputStreamout out) 메서드 : 출력 스트림에 쓸 데이터 압축
            > 압축되지 않은 데이터를 압축된 형태로 내부 스트림에 쓰는 CompressionOutputStream 생성
        - createInputSteam(InputStream in) 메서드 : 입력 스트림으로부터 읽어 들인 데이터 압축 해제
            > 기존 스트림으로부터 비압축 데이터를 읽을 수 있는 CompressionInputStream 반환
    - CompressionCodecFactory를 사용하여 CompressionCodec 유추
        - getCodec() 메서드 : 지정된 파일에 대한 Path 객체를 인자로 받아 파일 확장명에 맞는 CompressionCodec 찾아줌
        - removeSuffix() 정적 메서드 : 파일 접미사 제거
        - 나열된 코덱(LZO 제외) + io.compression.codecs 속성에 나열된 코덱을 불러옴
            > io.compression.codecs : 압축 및 해제를 위해 추가하고자 하는 CompressionCodec 클래스 목록
    - <sup>[2](#footnote_2)</sup>원시 라이브러리
        - 성능 관점에서 원시 라이브러리 사용하는 것이 바람직
        - 기본적으로 자신이 수행되는 플랫폼에 맞는 원시 라이브러리를 먼저 찾고, 있으면 자동으로 해당 라이브러리 로드
        - io.native.lib.avaliable = false : 원시 라이브러리 사용 비활성화
            > 내장되어 있는 자바와 동등한 코덱 라이브러리가 사용됨
        - 코덱 풀
            - 압축기와 해제기를 재사용해서 객체 생성 비용 절감 가능
            - 원시 라이브러리 사용 + 애플리케이션에서 상당히 많은 압축·해제 작업 수행시 사용

- 압축과 입력 스플릿
     - HDFS에 1GB 크기로 저장된 비압축 파일
        - 128MB 크기의 DHFS 블록 8개가 저장
        - 맵리듀스 잡은 개별적인 맵 태스크에서 독립적으로 처리되는 8개의 입력 스플릿 생성
    - 압축된 크기가 1GB인 단일 gzip 압축 파일
        - HDFS는 8개의 블록으로 저장
        - gzip 스트림이 특정 위치에서 읽기를 지원하지 않기에 각 블록별로 스플릿 생성 불가
            > 맵 태스크가 각 블록 스플릿을 개별적으로 읽는 것은 불가능
        - DEPLATE 압축 방식은 각 블록의 시작점을 구별할 수 없음 ∴ gzip은 분할 지원 X
        - 맵리듀스는 파일을 분할하지 않으면서 최적의 방식으로 작동 → 지역성 비용 발생
            > 단일 맵이 8개의 HDFS 블록 모두 처리 (대부분의 블록은 맵의 로컬에 있이 않을 것), 소수의 맵으로 잡이 덜 세분화되기에 많은 시간 소요
    - LZO
        - 내부 압축 포맷은 스트림과 동기화되는 방법을 리더에게 제공하지 않기에 같은 문제 발생
        - BUT, 색인도구 사용시 LZO 파일 전처리 가능
            > 색인도구 : 스플릿 지점의 색인을 구축하고 적당한 맵리듀스 입력 포맷으로 사용되어 파일을 효율적으로 분할할 것
    - bzip2
        - 블록 사이에서 동기화 표시자를 제공하여 분할 지원

※ 압축 포맷 선택
 - 파일 크기, 포맷, 초리에 사용되는 도구 같은 고려사항에 의해 결정  
 <효율성 순>  

    1. 압축과 분할 모두 지원하는 (시퀀스 파일, 에이브로, ORCFile, 파케이) 컨테이너 파일 포맷 사용, LZO, LZ4, Snappy와 같은 빠른 압축 형식이 적당 
    2. 분할 지원하는 압축 포맷 bzip2, 색인 될 수 있는 LZO 사용
    3. 애플리케이션에서 파일을 청크로 분할하고 지원 되는 모든 압축 포맷(분할 상관X)을 사용하여 각 청크를 개별적으로 압축
        > 청크 크기 = HDFS 블록 하나의 크기
    4. 압축하지 않고 그냥 저장
- 파일의 크기가 매우 크면 분할을 지원하지 않는 압축 포맷은 권장X
    > 지역성을 보장하지 않으면 상당히 비효율적

</br>

- 맵리듀스에서 압축 사용하기
    - mapreduce.output.fileoutputformat.compress = true : 맵리듀스 잡의 출력 압축
    - mapreduce.output.fileoutputformat.compress.codec : 사용할 압축 코덱의 클래스 이름 지정
    - mapreduce.output.fileoutputformat.compress.type : <sup>[3](#footnote_3)</sup>시퀀스 파일로 출력을 내보내고 있다면 사용할 압축 유형 제어 가능 
        > RECORD : 개별 레코드 압축 (default)  
        BLOCK : 레코드 그룹 방식으로 압축 (압축 성능 ↑) 
    - 맵 출력 압축
        - 압축되지 않은 데이터를 읽고 쓰더라도 맵 단계에서 임시 출력을 압축하면 이익 有
        - LZO, LZ4, Snappy와 같은 빠른 압축기를 사용하여 전송할 데이터양을 줄이면 성능 ↑

### 직렬화 
    - Serialization (직렬화) : 네트워크 전송을 위해 구조화된 객체를 바이트 스트림으로 전환하는 과정
    - Deserialization (역직렬화) : 바이트 스트림을 일련의 구조화된 객체로 역전환하는 과정

- 프로세스 간 통신, 영속적인 저장소와 같은 분산 데이터 처리시 나타남
- RPC (원격 프로시저 호출) 프로토콜
    - 하둡 시스템에서 노드 사이의 프로세스 간 통신
    - 원격 노드로 보내가 위한 메시지를 하나의 바이너리 스트림으로 구성하기 위해 직렬화 사용
    - 원격 노드에서 바이너리 스트림을 원본 메시지로 재구성하기 위해 역직렬화 사용
    - RPC 직렬화 포맷이 유익한 이유
        1. 간결성
            > 공간 효율적 사용, 네트워크 대역폭 절약
        2. 고속화
            > <sup>[4](#footnote_4)</sup>백본을 형성하기에 오버헤드 최소화
        3. 확장성
            > 새로운 요구사항 만족시키기 위해 발전에 직관적
        4. 상호운용성
            > 다양한 언어로 작성된 클라이언트 지원
    - 영속적인 저장소 포맷에서도 위의 항목은 중요
- Writable
    - 매우 간결하고 빠른 자체 직렬화 포맷 
    - 확장성과 상호운용성 지원 X

> 하둡은 RPC를 이용하여 클러스터 내의 노드들 간에 통신을 함  
이때 함수 인자나 리턴값들은 네트워크를 타고 송수신하는데 Writable 인터페이스 사용  

</br>

- Writable 인터페이스


</br></br></br></br></br></br></br>
---

<a name="footnote_1">1</a> : 파일의 비트 손상  
<a name="footnote_2">2</a> : primitive, 기본 제공~  
<a name="footnote_3">3</a> : 기본적으로 binary key/value 쌍을 저장하는 구조를 가지며 압축에 따라 다른 포맷으로 나눌 수 있음  
<a name="footnote_4">4</a> : 자신에게 연결되어있는 소형 회선들로부터 데이터를 모아 빠르게 전송할 수 있는 대규모 전송회선
