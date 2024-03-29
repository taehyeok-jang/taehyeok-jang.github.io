---
layout: post
title: (Paper Review) The Google File System - Part 1
subheading: 
author: taehyeok-jang
categories: [distributed-systems]
tags: [paper-review, distributed-systems, file-system]
---


## Background

Google 파일 시스템(GFS)은 대량의 데이터를 저장 및 처리하기 위한 Google의 분산 파일 시스템으로 2003년 SOSP '03 에서 처음 소개가 되었다. 지속적으로 개선이 되어 현재는 Collossus라는 이름으로 Google에서 사용되고 있으며 Google Cloud Platform (GCP)에서도 Collossus를 기반으로한 파일 시스템 서비스를 제공하고 있다.

세상에 나온지 거의 20년이 다 되어가는 논문이지만 대량의 데이터를 효율적이고 신뢰성 있게 저장하기 위한 GFS의 디자인 및 구현, 그리고 설계 과정에서의 고민은 오늘 날에도 여전히 유효하다. 논문을 읽으면서 분산 시스템이 가진 고유의 문제들을 해결하면서 효율적인 파일 시스템을 제공하기 위한 GFS의 여러 지혜를 얻을 수 있을 것이다.

GFS 논문 리뷰는 두 차례에 걸쳐 작성되었다. 간결하게 정리하고 싶었지만 논문에서 빠뜨릴 내용이 없어 대부분의 내용을 글에 담게 되었다. 



## Abstract

논문에서는 대량의 분산된 data를 처리하기 위한 scalable distributed file system인 Google File System(GFS)을 디자인 및 구현하였다. GFS는 commodity hardware을 기반으로 한 cluster에서 동작하며 fault tolerance하며, 다수의 client에게 높은 aggreagte performance를 보여준다. 

이전의 distributed file system의 목적을 공유하면서, GFS의 디자인은 google의 application workload와 technoloical enviroment의 현재와 미래 측면 모두를 반영하여 고안되었다. 이는 전통적인 file system을 다시 탐색하고 급진적인 다른 design points들을 탐험하게 하였다.

GFS는 google의 storage needs를 성공적으로 충족시켰다. 대량의 data set을 다루는 development, research 모두에서 data generation, processing을 위한 storage platform으로 사용되었다. 이 대량의 cluster는 수천 개의 machine 상에서 수백 TB의 data를 저장하고 수백개의 client로부터 동시 접근된다. 

논문은 다음 세가지를 담고있다. 

- file system extension designed to support distributed applications
- discuss many aspects of GFS's design 
- measurements from both micro-benchmarks and real world use



## 1. Introduction 

GFS는 google 내에서 빠른 속도로 증가하는 data processing에 대한 needs를 충족시키고자 탄생하였다. GFS의 디자인은 google의 application workload와 technoloical enviroment의 현재와 미래 측면 모두를 반영하여 고안되었다. 이는 전통적인 file system을 다시 탐색하고 급진적인 다른 design points들을 탐험하게 하였다.

첫째로, component failure는 예외 사항이 아니라 distributed system의 표준이다. 수천대의 machine으로 구성된 cluster는 저가의 상용 machine으로 구성되어 있으며 상응하는 수의 client로부터 access된다. machine의 quantity, quality가 의미하는 것은 언제라도 functional 하지 않을 수 있고 recove 되지 못할 수도 있다는 것이다. 우리는 application bug, operating system bugs, human erros, disk failure, memory, connector, network, power supplies 등 아주 다양한 곳에서 에러가 발생되는 것을 관찰하였다. 따라서 content monitoring, error detection, fault tolerance, automatic recovery는 system에 필수적이다.

둘째, 파일들은 전통적인 표준에 비해 그 크기가 매우 크다. 수 GB는 보통이다. 수 TB로 빠르게 증가하는 application 내 data를 처리할 때, 설령 file system이 지원 가능하다고 해도 수십억개의 KB-sized의 파일을 다루는 것은 매우 어렵다. 따라서 I/O operation과 block size 같은 design assumption 및 parameter는 다시 고려되어야 한다.

세번째, 대부분의 파일들은 기존의 data를 overwrite 하는 것 보다 새로운 data를 append 하는 것으로 mutate가 일어난다. random write 실질적으로 발생 빈도수가 적고 한번 write되면 대부분 read 혹은 sequential read만 발생한다. 대량의 파일에 대한 이러한 access pattern을 고려했을 때, append는 성능 최적화와 원자성 보장을 위한 focus가 되어야한다. 반면 client에 data block을 cache 하는 것은 설득력을 잃었다.

넷째, application과 file system API를 함께 설계하는 것은 flexibility를 높인다는 점에서 전체 system에 이득을 준다. 예를 들어 GFS는 file system을 매우 단순화하기 위해서 GFS의 consistency model을 단순화 하였다. 또한 다수의 client들이 추가적인 동기화 없이도 파일에 access 할 수 있도록 atomic append operation을 도입하였다. 

여러 GFS cluster들은 현재 다른 목적으로 배포되었다. 가장 큰 것은 300TB 이상의 disk storage와 1000개의 node를 가지고 있는데, 별도의 machine에 있는 수백개의 client로부터 지속적으로 heavily acccess 되고 있다. 



## 2. Design Overview 

### 2.1 Assumptions

file system을 설계하는 과정에서 우리는 도전과 기회를 모두 제공하는 가정을 따랐다. 우리는 이전에 몇 가지 key observations을 언급했으며 아래와 같다.

- inexpensive commodity hardware로 구성되어있으며 fail이 필연적으로 발생
- 시스템은 적당한 수의 large file을 저장한다. 100MB 이상 혹은 수 GB 파일을 저장하는 것이 common case
- workload는 두 종류의 read로 이루어져있다. large streaming read, small random read. 
- workload는 또한 파일에 data를 append하는 large, sequential write로 이루어져있으며 한번 write되면 거의 수정되지 않는다. 
- 시스템은 여러 client가 같은 파일에 대해 동시에 append 하는 것에 대한 잘 정의된 semantic이 필요하다.
- high sustained bandwidth가 low latency 보다 우선한다. google의 대부분의 application들이 대량의 data를 bulk로 high rate로 처리하는 것을 우선적으로 하며, 소수는 response time에 대한 조금 더 엄격한 요구사항이 있다. 



### 2.2 Interface

GFS는 친근한 file system interface를 제공하며, POSIX와 같은 standard API를 구현하지는 않았다. 파일들은 directory 내에 hierarchical하게 구성되며 path name으로 구별된다. 지원하는 interface는 다음과 같다.

- create, delete, open, close, read, and write files
- snapshot, record append

이중 record append는 다수의 클라이언트가 한 파일에 동시 접근하는 것을 허용하면서 각 클라이언트의 append에 대한 원자성을 보장한다. 이는 multi-way merge와 producer-consumer queue에 유용하다. 이러한 성질을 가진 파일은 large distribtued application을 구현하는 데 매우 유용하다. 



### 2.3 Architecture

[![img](https://1.bp.blogspot.com/-DPELgKlFiL0/X0Io6pJlnZI/AAAAAAAADio/5Ya6IwZOpAIrAVfAd7qNi2Mv6iKh3vWqwCLcBGAsYHQ/s640/GFS-Fig1.png)](https://1.bp.blogspot.com/-DPELgKlFiL0/X0Io6pJlnZI/AAAAAAAADio/5Ya6IwZOpAIrAVfAd7qNi2Mv6iKh3vWqwCLcBGAsYHQ/s1372/GFS-Fig1.png)

Fig 1은 GFS cluster를 나타낸다. GFS cluster는 하나의 master와 여러 chunkserver로 이루어져 있으며 여러 client에 의해 access되며 각각은 상용 linux machine에서 user-level server process로 동작한다.

file들은 고정된 크기의 chunk로 나뉘어지며, 각각의 chunk는 immutable, global한 64bit의 chunkhandle(id)를 가지며 chunk 생성 시에 master가 할당한다. reliability를 위해서 각 chunk는 여러 chunkserver에 replication된다. replication factor의 default는 3이지만 file namespace의 서로 다른 region에 대해서 임의로 replication level을 지정할 수 있다. 

master는 모든 file system의 metadata를 관리한다. 이는 namespace, access control 정보, file에서 chunk로의 mapping, chunk들의 현재 위치를 포함한다. master는 또한 chunk lease management, orphaned chunk에 대한 garbage collection, chunkserver 간 chunk migration 등 모든 system-wide activities를 control 한다.

GFS client는 file system API를 구현한 application과 연결되어 있으며 application을 대신하여 master, 각 chunkserver와 통신한다. master와 metadata operation을 수행하여 chunkserver와 data-bearing communication을 직접 수행한다. 

client와 chunkserver는 둘 다 file data를 caching하지 않는다. client cache는 거의 이점이 없는데 대부분의 application은 거대한 file을 단순히 stream하거나 혹은 cache하기에 너무 크기 때문이다. cache하지 않음으로써 cache coherence issue가 없게 되고 그 결과 client와 전체 system이 단순해진다. chunkserver는 file data를 cache할 필요가 없는데 chunk는 local file에 저장되어 있어 linux buffer cache는 이미 자주 access되는 data에 대해 memory에 cache를 하고 있다. 



### 2.4 Single Master

단일 master 구조는 디자인을 단순화시키고 master로 하여금 정교한 chunk placement와 global knowledge을 사용한 replication decision을 가능하게 하였다. 하지만 master가 read/write에 최소한으로 개입하여 master를 통한 bottleneck을 방지해야 한다. client는 master를 통해 read/write 하지 않으며, 대신에 client는 master로부터 data를 주고받기 위해 어떤 chunkserver와 접촉해야 하는지에 대한 정보를 얻는다. Fig 1는 read를 하기 위해서 client와 master, chunkserver가 어떤 방식으로 interact 하는지 나타냈다.



### 2.5 Chunk Size

chunk size는 핵심 design parameter 중 하나이다. GFS에서는 64MB로 정하였는데, 보통 file system의 block size보다 아주 크다. chunk replica는 각 chunkserver에 plain linux file로 저장되며 필요한 경우에만 확장된다. lazy space allocation은 internal fragmentation으로 인한 space 낭비를 방지한다.

큰 chunk size는 아래와 같은 이점을 지닌다.

1) read/write가 동일한 chunk에 주로 발생하게 함으로써 master와의 interaction을 줄인다. 이러한 효과는 read/wrtie가 sequential 하게 발생하는 application에서 극대화된다.

2) 동일한 chunkserver에 접근하면서 TCP connection을 persistent하게 유지하여 network overhead를 줄인다. 

3) master가 가지고 있어야 할 metadata 크기를 줄일 수 있고 memory에 적재함으로써 여러 이점을 얻는다. 

반면 lazy space allocation에도 불구하고 한계점도 가지고 있다. application에서 다수의  client가 어떤 특정 chunk에 대한 access rate가 높아지면 해당 chunkserver는 hot spot이 된다. GFS에 대한 테스트를 수행하면서 이 문제가 발견되었는데 해당 파일에 대한 replication factor를 높이고 application 수행 시점을 분산시킴으로써 병목을 완화함으로써 해결하였다. 



### 2.6 Metadata

master는 세가지 주요 metadata를 저장한다.

1) file and chunk namespaces

2) mapping from files to chunks

3) locations of each chunk's replicas

이중에서 위 두개는 memory에 저장되면서 또한 mutation에 대한 operation log를 local disk, remote machine에 persistent하게 저장한다. operation log를 남김으로써 master update를 단순하고 신뢰성있고 일관성있게 할 수 있다. 반면 chunk replica에 대한 location은 persistent 하게 저장하지 않고 chunkserver가 시작하면서부터 그에 대한 정보를 주기적으로 반영한다. 



#### 2.6.1 In-Memory Data Structure

metadata를 memory에 저장함으로써 master operation은 신속하게 이루어질 수 있다. 게다가 background에서 master의 상태를 주기적으로 scan하기에도 유리하다. 이 주기적인 scan은 chunk garbage collection, chunkserver failure에 따른 re-replication, load과 disk usage의 balance를 위한 chunk migration을 위해 이루어진다. 

memory-only approach에 대한 잠재적인 문제로 chunk의 개수가 많아지면서 memory capacity 부족이 있을 수 있다. 하지만 practical 하게 전혀 문제가 되지 않는데 64MB chunk를 위해서 master는 단순히 64 bytes가 되지않는 metadata를 저장하고 있기 때문이다. 

정말로 memory 확장이 필요하다면 이 대량의 cluster에 대해서 memory를 scale up 하는 것은 그다지 문제가 되지 않는다.



#### 2.6.2 Chunk Locations

master는 어떤 chunkserver가 chunk에 대한 replica를 가지고 있는지 persistent 하게 저장하지 않는다. master는 단순히 chunkserver의 startup부터 poll 하고 있으며 주기적인 heartbeat를 보내어 상태를 monitor 및 update한다. 

GFS 설계 초기에는 persistent하게 저장하려고 하였으나, 위와 같은 poll 방식이 훨씬 단순하게 접근하다고 결정하였다. 이는 master와 chunkserver 간 sync 문제를 제거했다.

이 design을 더 단순하게 이해하는 다른 방식은 어떤 chunk에 대한 final word, 즉 최종 책임자는 chunkserver라는 것을 깨닫는 것이다. 이 같은 방식에서는 chunkserver 내에서도 얼마든지 실패할 수 있기에 master가 일관된 view를 가지고 있어야 할 지점이 없다.



#### 2.6.3 Operation Log (*) 

operation log는 중요한 metadata 변경에 대한 historical record이며 GFS의 핵심이다. operation log는 metadata의 persistent record이자 cocurren operation에 대한 논리적인 시간 순서를 제공한다.

operation log는 중요하기 때문에 신뢰성 있게 저장되어야 하며 metadata가 persistent하게 저장되기 전까지 client에 보여지면 안된다. 여러 remote machine에도 저장되며 해당 log가 local disk 및 remote machine에 모두 flush 한 다음에 client에 응답한다.

master는 file system을 operation log를 replay함으로써 복원한다. 시작 시간을 줄이기 위해 log를 작게 유지해야 하는데, master는 operation log 크기가 커질 때마다 checkpoint를 갱신하여 가장 최신의 checkpoint를 load 하고 남아있는 log record를 실행하여 이를 가능하게 한다. checkpoint는 compact B-tree에 있어 신속한 접근이 가능하다.

checkpoint를 생성하는 데는 시간이 소요되므로 master의 내부 상태는 새로운 checkpoint가 incoming mutations에 대한 지연이 없이 생성될 수 있도록 구성되어 있다. master는 새로운 log file로 별도의 thread를 통해 switch 하며 이 log file은 checkpoint 이전 시점에 발생한 모든 mutation을 반영한다. 



### 2.7 Consistency Model (*) 

GFS는 google의 distributed application을 지원하기 위해 완화된 consistency model을 가지고 있다. 우리는 GFS의 guarantee와 이것이 application에 무엇을 의미하는지 이야기 할 것이다. 

#### 2.7.1 Guarantee by GFS

[![img](https://1.bp.blogspot.com/--glHtB9TGaE/X0Ips-BKr0I/AAAAAAAADjA/A4vMGB2aASwwUfSYeUWMRImUHcBNE8UzACLcBGAsYHQ/w410-h180/GFS-Tab1.png)](https://1.bp.blogspot.com/--glHtB9TGaE/X0Ips-BKr0I/AAAAAAAADjA/A4vMGB2aASwwUfSYeUWMRImUHcBNE8UzACLcBGAsYHQ/s820/GFS-Tab1.png)

file namespace mutation. 

file namespace mutation은 atomic 하며 master에 의해 exclusive 하게 이루어진다. namespace locking은 atomicity와 correctness를 보장하며 master operation log는 이들 operation의 전체 순서를 정의한다.

data mutation 이후의 file region의 상태는 mutation의 type, 성공 여부, cocurrent 여부에 따라서 달라진다. table 1에서 이를 나타내고 있다.

consistent: all clients will see always the same data.

defined: consistent + reflect all cocurrent mutations. 

전형적으로, concurrent mutations에 따른 data는 mutations이 적용된 일부분으로 이루어져있다. 실패한 mutation은 region을 inconsistent하게 만들며 서로 다른 client가 다른 data를 볼 수 있다. 아래에서 application이 defiend region과 undefined region을 어떻게 구분하는지 살펴보겠다. 

data mutations.

write: write data at an application-specified file offset.

record append: append data atomically at least once even in the presence of concurrent mutations, but at an offset of GFS's choosing.

일련의 mutations가 발생한 다음에도 대상 mutate file region은 defined가 보장되며 마지막 mutation에 의한 data를 가지고 있다. GFS는 다음의 방법으로 이를 가능하게 한다.

a) chunk에 대한 mutations을 모든 replica에 동일한 순서로 적용한다.

b) chunk version number을 사용하여 chunkserver의 fail에 따른 stale한 상태의 chunk를 확인한다. 

client는 chunk location을 cache하고 있기 때문에 stale replica로부터 read를 할 수 있다. 이는 cache entry의 timeout 만큼, 그리고 다음 file을 open 하기 전까지로 한정된다. 그리고 대부분의 파일이 append-only이기 때문에 outdated data를 보기보다는 prematured end를 return 한다. (따라서 application의 부작용에 대한 영향도도 제한된다는 의미로 해석)

여러 mutation 이후에 component failure는 data는 corrupt 혹은 destroy 시킬 수 있다. GFS는 master와 chunkserver 간 주기적인 handshake로 이를 확인하며 checksum을 통해 data corruption을 detect 한다.



#### 2.7.2 Implications For Applications

GFS application은 몇가지 단순한 technique으로 완화된 cosistency model을 수용할 수 있다. 이 technique들은 다른 목적을 위해 이미 필요한 것들이며 아래와 같다.

relying on appends rather than overwrites, checkpointing, writing self-validating, self-identifying record.

전형적인 사용 예로서, 단일 writer가 file을 시작부터 끝까지 다 생성한다. data를 모두 write 한 다음 atomic하게 파일 이름을 변경하거나, 주기적으로 얼만큼 write 되었는지 주기적으로 checkpoint한다. checkpoint는 application-level checksum을 포함하여 reader는 마지막으로 defined state가 된 file까지만 읽는다. 

다른 전형적인 사용 예로, 여러 writer가 merged result 혹은 producer-consumer를 위해 동시에 file에 append 하는 경우가 있다. record append의 at-least-once semantic은 writer의 ouput을 보존하며 reader는 그에 따른 padding이나 duplicate를 다룬다. 각 record에는 verification을 위한 checksum 등이 준비될 수 있으며 reader는 application에서 padding, duplicate를 filtering 할 수도 있다. 



## 3. System Interactions

GFS는 master의 개입을 최소한으로 하는 system을 디자인되었으며 client, master, chunkserver가 어떻게 상호작용하는지 살펴보자.

[![img](https://1.bp.blogspot.com/-GniHhMBsJRY/X0IqFMEg00I/AAAAAAAADjI/sQdbRu9z6lYCUQkZ1--X7kd3IHBwmockgCLcBGAsYHQ/s640/GFS-Fig2.png)](https://1.bp.blogspot.com/-GniHhMBsJRY/X0IqFMEg00I/AAAAAAAADjI/sQdbRu9z6lYCUQkZ1--X7kd3IHBwmockgCLcBGAsYHQ/s834/GFS-Fig2.png)

### 3.1 Leases and Mutation Order

mutation은 chunk의 contents나 metadata를 변경시키는 동작이다. 각 mutation은 chunk의 replica 모두에 적용된다. GFS에서는 여러 replica에 일관성 있는 mutation order를 적용하기 위해서 lease 개념을 도입하였다. master는 chunk의 한 replica에 lease를 부여하며 이 replica는 primary라 불린다. primary에서는 mutation들에 serial number를 적용하며 나머지 replica인 secondary는 모두 이 순서를 따른다. lease 메커니즘은 master의 management overhead를 최소화하기 위해 디자인되었다. 초기 timeout은 60초이지만 chunk가 mutate 되고 있는 동안에는 무기한으로 늘어날 수 있다. 이를 위한 신호는 master, chunkserver 간 heartbeat에 piggyback 된다. 

Figure 2는 write를 위한 control flow를 나타낸다.

1) client는 master에게 현재 lease를 가지고 있는 chunkserver와 (primary) 다른 replica의 location 정보(secondary)를 요청한다. lease가 아직 부여되지 않았으면 replica들 중 하나에 부여한다.

2) master는 client에 primary, secondary에 대한 정보를 반환한다.

3) client는 모든 replica에 data를 push 한다. 각 chunkserver는 data가 저장되거나 age out 될 때까지 내부 LRU buffer cache에 저장한다. data flow와 control flow를  decouple 함으로써 값비싼 data flow에 대한 scheduling을 함으로써 performance를 증대한다.

4) 모든 replica가 ack를 보내면 client는 write request를 보낸다. primay는 연속적인 mutations들에 serial number를 할당하고 그 순서에 맞게 적용한다.

5) priamry는 secondary에 write request를 forward하고 모든 secondary는 그 순서에 맞게 mutations을 적용한다. 

6) 모든 secondary가 primary에 응답하면 operation을 완료한 것이다.

7) primary는 client에 응답한다. 어떤 에러가 발생하여 clien 실패가 발생할 수 있다. 실패한 region은 primary 및 secondary의 부분 집합일 수 있으며 수정된 region에서는 inconsistent한 상태가 된다. client는 retry를 통해 실패한 mutations 부터 다시 write를 시도한다.

write가 용량이 클 때는 여러 write operation으로 나누어서 처리한다. 모든 replica에서 각 operation을 잘 처리하여서 identical 하더라도 공유된 file region은 fragment를 가지고 있을 수 있다. 이는 file region이 consistent 하지만 undefined state에 있게한다.  

### 3.2 Data Flow

우리는 network를 효율적으로 활용하기 위해서 data flow와 contorl flow를 분리하였다. control flow는 client에서 primary로, 그다음 secondary로 전되는 반면, data flow는 잘 채택된 chunkserver의 pipelined chain에 선형적으로 push된다. 목적은 각 machine의 bandwidth의 사용을 극대화하고, network bottleneck과 high-latency link를 피하고 모든 data를 push하는 데 필요한 latency를 최소화하는 것이다.

또 TCP connection 상에서 data trasfer를 pipeline 함으로써 latency를 최소화했다. chunkserver가 data를 일단 받으면 바로 chaining된 replica에 바로 forward를 시작한다. network 혼잡이 없다는 가정 하에 이상적인 elapse time은 아래와 같다.

B/T + RL (B: transferred bytes, T: network throughput, R: replicas, L: latency between two machines)



### 3.3 Atomic Record Appends

GFS는 record append라는 atomic append 동작을 지원한다. traditional write에는 client가 offset을 명시적으로 지정해야 하고, 따라서 같은 region에 대한 concurrent write가 불가능하다. 반면 record append는 client는 offset 없이 data만 지정을 한다. GFS는 data를 GFS가 정한 offset에 at-leas-once, atomic 하게 append 한다. client는 여러 client들 간 동기화 없이도 같은 file에 대해서 atomic하게 record append 하는 것이 가능하다.

record append가 어떤 replica에서 fail 한다면 client는 해당 동작을 retry 할 것이다. 그 결과 같은 chunk에 대한 replica들은 duplicate를 포함한 서로 다른 data를 가지고 있을 수 있다. GFS는 모든 replica가 bytewise로 같을 것을 보장하지 않는다. 단지 data가 atomic unit에 대해 at-lease-once로 write 되는 것을 보장한다. 우리의 consistency 보장에 따르면 성공적으로 record append한 region은 defined 상태이고 중간에 intervene 된 region은 inconsistent 상태이다. application에서는 2.7.2에서 언급한대로 inconsistency를 다룰 수 있다. 



### 3.4 Snapshot

snapshot 동작은 file이나 directory tree의 copy를 만들며, 이는 진행 중인 mutations에 대한 방해를 최소화하면서 거의 즉시 이루어진다.

AFS처럼 GFS에서는 snapshot을 구현하기 위해 copy-on-write technique이 사용된다. snapshot은 다음 순서로 이루어진다.

1) master가 snapshot 요청을 받으면 먼저 snapshot의 target이 되는 chunk의 lease를 revoke 시킨다. 이는 그 다음 들어오는 write가 master와 interaction 하도록 강제하며,  master가 새로운 copy를 만들 수 있을 기회를 준다.

2) lease가 revoke 혹은 expire되면 master는 disk에 이 동작을 log로 남긴다. 그 다음 source file이나 directory tree의 metadata를 복제함으로써 이 log를 in-memory 상태에 적용한다. 

3) client에서 current lease holder에 대한 요청이 올 때 master는 chunk C에 대한 reference count가 1보다 큼을 알아차린다. master는 client의 요청을 defer 시키며 새로운 chunk C'를 선택한다. 

4) 그다음 각 chunk C를 들고 있는 chunkserver에게 새로운 C'를 만들라 요청한다. C'을 local에서 복제함으로써 아주 빠르게 이루어진다. 

5) 이 시점에서 request handling은 다른 요청과 다르지 않다. master는 C'에 대해 lease를 grant 함으로써 client가 C'에 대해서 아무런 인지 없이 정상적으로 사용할 수 있도록 한다. 



## References 

- Sanjay Ghemawat, Howard Gobioff, & Shun-Tak Leung (2003). The Google File System. In *Proceedings of the 19th ACM Symposium on Operating Systems Principles* (pp. 20–43).

- https://courses.cs.washington.edu/courses/cse490h/11wi/CSE490H_files/gfs.pdf
- https://cs.stanford.edu/~matei/courses/2015/6.S897/slides/gfs.pdf



