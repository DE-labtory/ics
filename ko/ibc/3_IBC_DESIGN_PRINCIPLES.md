# 2: IBC 디자인 원칙

**이것은 IBC의 "디자인 원칙"에 대한 설명입니다.**

**IBC에서 사용되는 용어들의 정의는, [여기](./1_IBC_TERMINOLOGY.md)를 참조 하십시오.**

**설계 개요는 [여기](./2_IBC_ARCHITECTURE.md)를 참조 하십시오.**

**예제 use case는, [여기](./4_IBC_USECASES.md)를 참조 하십시오.**

**디자인 패턴에 대한 설명은 [여기](./5_IBC_DESIGN_PATTERNS.md)를 참조 하십시오.**

"블록체인 간 통신 프로토콜"의 설계 범위는 매우 넓고, 용어 자체로도 많은 것을 포괄합니다. "블록체인 간 커뮤니케이션 프로토콜" (IBC)은 상호 운용 가능한 블록체인의 인터체인 환경에서 다양성, 지역성, 모듈성 및 효율성을 제공하기 위한 프로토콜을 뜻합니다. 이 문서에서는 IBC를 사용하는 "이유"에 대해 간략히 설명하고, 기본적인 상위 레벨의 디자인 목표에 대해 세세하게 알아보도록 하겠습니다.

## 다양성 (Versatility)

IBC는 *가변적인* 프로토콜을 지향합니다. 이 프로토콜은 서로 다른 언어로 구현되고, 다른 의미를 갖는 *이종의* 블록체인을 지원합니다. IBC 위에서 작성된 애플리케이션은 함께 *구성될 수 있으며*, IBC 프로토콜의 단계들은 *자동화 될 수 있습니다.*

### 이질성 (Heterogeneity)

IBC는 기본적인 요구사항(빠른 완결성, 임의의 합의 알고리즘과 상태 머신, 상수 시간복잡도의 상태 커밋, 간결한 증명)들을 만족하는 합의 알고리즘과 상태 머신으로 구성될 수 있습니다. 프로토콜은 멀티 체인 애플리케이션에서 일반적으로 요구하는 사항인 데이터 검증, 전송, 정렬에 대해서 다루지만, 애플리케이션 그 자체의 의미와는 무관합니다. IBC로 연결된 서로 다른 두 개의 체인은 반드시 애플리케이션 층에서 호환되는 "인터페이스" (토큰 전송 방법과 같은)에 대해 알고 있어야 하지만, IBC 인터페이스 핸들러를 거치면서 상태 머신은 임의의 맞춤형 기능(보호된 트랜잭션과 같은)을 제공할 수 있습니다.

### 구성가능성 (Composability)

IBC 위에서 작성된 여러 애플리케이션은 각 프로토콜 개발자와 사용자에 의해 함께 구성될 수 있습니다. IBC는 인증, 전송, 정렬과 관련된 기본적인 요소들과, 자산과 데이터 시멘틱을 위한 애플리케이션 레이어 기준들을 정의합니다. 호환 가능한 기준들을 제공하는 체인들은 커넥션을 연 유저 (또는 커넥션을 재사용하는 유저)들에 의해 서로 연결되고, 거래에 사용될 수 있습니다. 자산과 데이터는 여러 체인들에 자동("멀티-홉"), 또는 수동으로(여러 IBC에 릴레이된 트랜잭션들을 순서대로 전송하며) 전송 될 수 있습니다.

### 자동성 (Automatability)

IBC에서 연결을 설정하고, 채널을 만들며, 패킷을 전송하고 비잔틴을 보고 하는 등의 역할을 하는 "사용자" 또는 "행위자"는, 사람일 수도 있지만 꼭 사람일 필요는 없습니다. 모듈들, 스마트 컨트랙트, 자동화된 off-chain 프로세스들은 프로토콜(예를 들어, 가스 비용 계산)을 사용하여 자체적으로나 공동으로 역할을 담당할 수 있습니다. 여러 체인에서 발생하는 복잡한 상호 작용들은(예를 들어 3단계 연결 설정을 위한 handshake, 또는 다중 hop 토큰 전송) 초기화 작업을 제외한 모든 것들을 사용자로부터 추상화 할 수 있도록 설계되었습니다. 결과적으로, 새로운 블록체인(모듈로 물리적 인프라 프로비져닝)을 자동으로 가동하고, IBC 연결을 시작하며 새로운 상태 머신과 성능을 자동으로 사용할 수 있습니다.

## 모듈성 (Modularity)

IBC는 *모듈로 구성된* 프로토콜로 디자인 되었습니다. 이 프로토콜은 명확한 보안 속성들과 요구사항을 만족하는 계층 구조로 구성되었습니다. 특정 층의 컴포넌트 구현은 상위 계층에 제공하는 여러 필수 요구사항(예를 들어, 서로 다른 합의 알고리즘, 또는 연결 설정 단계)에 따라 달라질 수 있습니다.(예를 들어 확정성, < 1/3 비잔틴 안전성, 또는 두 체인에 포함된 신뢰할 수 있는 상태). 상태 머신은 안전한 상호 작용을 위해 오로지 호환성 있는 IBC 프로토콜의 서브셋(예를 들어, 상호간의 합의를 위한 가벼운 클라이언트 검증 알고리즘)만을 이해하면 됩니다.

## 지역성 (Locality)

IBC는 로컬 프로토콜로 디자인 되었으며, 이는 단지 서로 연결된 두 체인은 보안과 양방향 IBC 연결에 대한 정확성에 대해서만 알 필요가 있다는 뜻입니다. 인증을 위한 보안 요구사항들은 연결된 블록체인의 합의 알고리즘과 검증자 집합에만 관련 있으며, IBC 연결들의 집합을 유지하는 블록체인은 서로 어떤 체인에 연결되어있는 것과 상관 없이, 연결된 블록체인의 상태만을 알고 있으면 됩니다.

### 정보와 커뮤니케이션의 지역성

IBC는 네트워크 토폴로지의 구조에 의존하지 않습니다. 글로벌 블록체인들의 네트워크 토폴로지는 고려할 필요가 없습니다: 보안 및 정확성은 두 체인간의 단일 연결 수준에서 토폴로지의 하위 그래프를 추론할 수 있습니다. 유저들과 체인들은 그들이 신뢰하고 있는 블록체인의 일부 정보만을 이용하여 그들의 가정과 리스크를 추론할 수 있습니다.

IBC에서는 "루트(root)체인"이 필요 없습니다. 글로벌 네트워크의 몇몇 서브 그래프들은 허브-스포크 구조로 진화할 수도 있고, 다른 네트워크들은 단단하게 결합된 채로 유지될 수도 있습니다. 또 다른 네트워크들은 더욱 이국적인 토폴로지를 유지할 수도 있습니다. 채널들은 점대점 연결로 동작합니다. IBC의 첫번째 버전은 오직 하나의 hop paths를 지원하지만, 미래에는 다중 hop paths를 지원할 것입니다.(연관된 합의 알고리즘의 정확성 가정으로 인해 자동 라우팅이 반드시 필요하거나 안전하지 않을수도 있습니다.)

그러나 애플리케이션의 데이터는 복잡한 다중 hop 경로로 전송되었을 수도 있는 토큰의 기존 영역, 원래 지분 또는 검증자가 크로스 체인 검증에 제공하는 그들의 신원, 또는 대체 불가능한 토큰을 관리하는 키와 관련된 스마트 컨트랙트와 같은 지역적이지 않은 속성들을 갖고 있을 것입니다. 이런 지역적이지 않은 속성들을 IBC 프로토콜이 이해할 필요는 없지만, 유저나 상위 레벨에 애플리케이션들에 의해 추론될 필요가 있습니다.

### 정확성 가정 & 보안의 지역성

IBC를 사용하는 유저는(사람 또는 스마트 컨트랙트) 어떤 합의 알고리즘을 사용할지, 어떤 상태 머신을 사용할지, 그들이 "옳다고 믿는" 검증자를 어떻게 선택할지 정해야 합니다. IBC 프로토콜이 올바르게 구현되었다고 가정하면, 유저들은 비잔틴 공격이나 잘못된 상태 머신 전환 등에 의해 발생하는 애플리케이션 레벨에서의 위험(자산 인플레이션과 같은) 등에 노출되지 않습니다. 이 특징은 미래의 거대한 인터체인 블록체인 네트워크 토폴로지와, 검증자 집단이 비잔틴 행위를 할 가능성이 있다는 점에서 매우 중요합니다. 보수적으로 구현된 IBC는 이런 위험을 제한하고 발생할 수 있는 피해를 줄일 수 있습니다.

### 허용 지역성

연결을 열고, 채널을 만들고 패킷을 전송하는 등의 IBC의 행동들은 상태 머신들과 연결된 특정 두 체인에 의해 지역적으로 허용됩니다. 독립적인 체인들은 애플리케이션 레벨의 행동들에 대한 권한 부여 메커니즘(합의와 같은)의 승인을 요구할 수 있지만, 기본적인 프로토콜에서 대부분의 행위는 모듈러 가스, 비용 부과 정책 등에 의해 제한됩니다. 기본적으로 커넥션 연결과 채널 생성은 가능하고, 패킷 또한 허용 프로세스 없이 전송될 수 있습니다. 물론, 유저들은 반드시 각 IBC 연결의 상태와 합의에 대해 점검해야 하고, 그것들이 충분히 안전한지 결정해야 합니다(신뢰할 수 있는 상태 저장과 같은 방법을 이용하여).

## 호율성

IBC는 *효율적인* 프로토콜로 설계되었습니다: 데이터 및 자산의 릴레이 중 발생하는 상각 비용은 주로 패킷과 연관된 기본 상태 전이 또는 운영 비용 (예 토큰을 전송하는 등), 일부 작은 상수 시간의 오버헤드로 구성되어야합니다.