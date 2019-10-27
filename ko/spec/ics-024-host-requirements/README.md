---
ics: '24'
title: 호스트 상태 머신 요구사항
stage: 초안
category: IBC/TAO
requires: '23'
required-by: 2, 3, 4, 5, 18
author: 'Christopher Goes '
created: '2019-04-16'
modified: '2019-08-25'
---

## 개요

이 표준 문서는 블록체인간 통신 프로토콜 구현체를 호스팅하는 상태머신이 반드시 제공하거나 충족되어야 하는 최소한의 인터페이스의 묶음을 정의합니다.

### 동기(Motivation)

IBC는 다양한 종류의 블록체인 및 상태 머신에 의해  호스팅되는 공통 표준으로 설계되었으며 호스트의 요구사항을 명확하게 정의해야 합니다.

### 정의

### 원하는 속성

IBC는 정확한 구현의 용이성을 최대화하기 위해 기본 상태 머신으로부터 가능한 간단한 인터페이스를 요구해야 합니다.

## 기술적 명세

### 모듈 시스템

호스트 상태 머신은 반드시 모듈 시스템을 지원하며(이를 통해 독립적이고, 잠재적으로 상호 신뢰할 수 없는 코드 패키지가 동일한 원장에서 안전하게 실행될 수 있습니다) 다른 모듈과 통신 할 수 있는 방법과 시기를 제어하고 "마스터 모듈" 또는 실행 환경에 의해 식별 및 조작 되어야 합니다.

IBC/TAO 사양은 핵심적인 "IBC handler" 모듈과 "IBC relayer"모듈의 구현을 정의합니다. IBC/APP 사양은 어플리케이션 로직을 처리하는 특정 패킷을 위한 다른 모듈을 추가로 정의합니다. IBC는 "마스터 모듈" 또는 실행 환경을 사용하여 호스트 상태 머신의 다른 모듈에 IBC 핸들러 모듈 및/또는 IBC 라우팅 모듈에 대한 액세스 권한을 부여 할 수 있지만, 그렇지 않으면 상태 머신에 공존 할 수 있는 다른 모듈의 기능 또는 통신 기능에 대한 요구사항을 부과하지 않습니다.

### Paths, identifiers, separators

`Identifier` 는 커넥션, 채널 또는 라이트 클라이언트와 같이 상태에 저장된 객체의 키로 사용되는 바이트 문자열입니다. 식별자는 영숫자로만 구성되어야 합니다. 식별자는 비어있지 않아야 합니다(양의 정수 길이).

`Path` 는 상태에 저장된 객체의 키로 사용되는 바이트 문자열입니다. 경로는 Identifier, 상수 영숫자 문자열 및 separators `"/"` 만 포함해야 합니다.

Identifiers는 가치있는 자원으로 의도된 것이 아니며 name squatting을 방지하기위해, 최소 길이 요구 사항 또는 의사 난수 생성이 구현 될 수 있지만 특정 제한이 적용되지 않습니다.

`"/"` 구분 기호는 두 개의 identifier 또는 identifier와 상수 바이트 문자열을 분리하고 연결하는 데 사용됩니다. identifier는 모호성을 방지하는 `"/"` 문자를 포함해서는 안됩니다.

중괄호로 표시되는 변수 보간(variable interpolation)은 이 사양 전체에서 경로 형식을 정의하는 속기로 사용됩니다. 예시, `client/{clientIdentifier}/consensusState`.

### Key/value Store

호스트 상태 머신은 표준 방식으로 작동하는 세 가지 기능을 갖춘 key/value store 인터페이스를 제공해야합니다.

```typescript
type get = (path: Path) => Value | void
```

```typescript
type set = (path: Path, value: Value) => void
```

```typescript
type delete = (path: Path) => void
```

`Path`는 위에서 정의한 대로 입니다. Value는 특정 데이터 구조를 인코딩한 임의의 바이트 문자열 입니다. 인코딩 세부 사항은 ICS를 분리하기 위해 남겨집니다.

이러한 함수는 IBC 핸들러 모듈 (구현은 별도의 표준으로 설명 됨)에만 권한이 있어야 하므로, IBC 핸들러 모듈만 path(`get`을 통해  조회될 수 있는)를 `set` 하거나 `delete`할 수 있습니다. 이것은 전체 상태 머신이 사용하는 더 큰 key/value store의 하위 store(접두사 key를 가지는)로 구현 될 수 있습니다.

호스트 상태 머신은 반드시 이 인터페이스의 두 인스턴스를 제공해야합니다.
다른 체인이 읽을 (즉, 입증 된) 스토리지를위한 `provableStore` 와 `get`, `set` 및   `delete`를 호출 할 수 있는 호스트에 로컬로 저장하기위한 `privateStore `. (예: `provableStore.set('some/path', 'value')`.

The `provableStore`:

- [ICS 23](../ics-023-vector-commitments)에 정의 된 vector commitment로 외부에서 입증 할 수 있는 데이터를 key/value store에 기록해야합니다.
- 이 사양에 제공된 표준 데이터 구조 인코딩을 proto3로 사용해야합니다.

The `privateStore`:

- 외부 증명을 지원할 수도 있지만 반드시 필요한 것은 아닙니다. IBC 핸들러는 증명해야 할 데이터를 절대로 쓰지 않습니다.
- 표준 proto3 데이터 구조를 사용할 수도 있지만 반드시 필요한 것은 아닙니다. 응용 프로그램 환경에서 선호하는 형식을 사용할 수 있습니다.

> 참고 : 이러한 방법과 속성을 제공하는 어떠한 키/값 저장소 인터페이스도 IBC에 사용되기에 충분합니다. 호스트 상태 머신은 스토어 인터페이스를 통해 설정되고 검색된 경로 및 값 쌍과 직접 일치하지 않는 경로 및 값 맵핑을 사용하여 "프록시 저장소"를 구현할 수 있습니다. 경로는 단일 페이지에서 입증 될 수있는 페이지에 저장된 버킷 및 값으로 그룹화 될 수 있습니다. `get` , `set` 및 `delete` 가 예상대로 동작하고 다른 기계가 검증 가능한 경로 및 값 쌍 (또는 그 부재)에 대한 커밋 증명을 확인 할 수 있는 한, 약정 방식으로 경로 공간을 일부 연속적인 방식으로 비 연속적으로 다시 맵핑 할 수 있습니다. 해당되는 경우, 스토어는 반드시 맵핑을 외부에 노출시켜 클라이언트 (relayer 포함)가 스토어 레이아웃 및 증거 구성 방법을 결정할 수 있습니다. 이러한 프록시 저장소를 사용하는 머신의 클라이언트도 맵핑을 이해해야 하므로, 새로운 클라이언트 유형 또는 매개 변수화된 클라이언트가 필요합니다.

> 참고: 이 인터페이스는 특정 스토리지 백엔드 또는 백엔드 데이터 레이아웃이 필요하지 않습니다. 상태 머신은 저장소가 지정된 인터페이스를 충족하고 commitment proof를 제공하는 한 필요에 따라 구성된 스토리지 백엔드를 사용하도록 선택할 수 있습니다.

### Path-space

현재 IBC/TAO는 `provableStore` 및 `privateStore`에 다음 경로 접두사를 사용합니다.

향후 Path는 향후 버전의 프로토콜에서 사용될 수 있으므로, 제공 가능한 저장소의 전체 key-space는 반드시 IBC handler를 위해 예약되어야합니다.

개인 저장소의 일부는 IBC 핸들러가 필요한 특정 키에 독점적으로 액세스 할 수 있는 한 다른 목적으로 안전하게 사용될 수 있습니다.
개인 저장소에 사용 된 키는 여기에 정의 된 키 형식과 개인 저장소 구현에 실제로 사용 된 형식간에 이분 매핑이 존재하는 한 안전하게 변경 될 수 있습니다.

Store | Path format | Value type | Defined in
--- | --- | --- | ---
privateStore | "clients/{identifier}" | ClientState | [ICS 2](../ics-002-client-semantics)
provableStore | "clients/{identifier}/consensusState" | ConsensusState | [ICS 2](../ics-002-client-semantics)
provableStore | "clients/{identifier}/type" | ClientType | [ICS 2](../ics-002-client-semantics)
provableStore | "connections/{identifier}" | ConnectionEnd | [ICS 3](../ics-003-connection-semantics)
privateStore | "ports/{identifier}" | CapabilityKey | [ICS 5](../ics-005-port-allocation)
provableStore | "ports/{identifier}/channels/{identifier}/key" | CapabilityKey | [ICS 4](../ics-004-channel-and-packet-semantics)
provableStore | "ports/{identifier}/channels/{identifier}/nextSequenceRecv" | uint64 | [ICS 4](../ics-004-channel-and-packet-semantics)
provableStore | "ports/{identifier}/channels/{identifier}/packets/{sequence}" | bytes | [ICS 4](../ics-004-channel-and-packet-semantics)
provableStore | "ports/{identifier}channels/{identifier}/acknowledgements/{sequence}" | bytes | [ICS 4](../ics-004-channel-and-packet-semantics)
privateStore | "callbacks/{identifier}" | ModuleCallbacks | [ICS 26](../ics-026-routing-module)

### 모듈 레이아웃

공간적으로 표현하면 호스트 상태 머신에서 모듈의 레이아웃 및 포함 관계 사양은 다음과 같습니다. (Aardvark, Betazoid 및 Cephalopod는 임의의 모듈 임)

```
+----------------------------------------------------------------------------------+
|                                                                                  |
| Host State Machine                                                               |
|                                                                                  |
| +-------------------+       +--------------------+      +----------------------+ |
| | Module Aardvark   | <-->  | IBC Routing Module |      | IBC Handler Module   | |
| +-------------------+       |                    |      |                      | |
|                             | Implements ICS 26. |      | Implements ICS 2, 3, | |
|                             |                    |      | 4, 5 internally.     | |
| +-------------------+       |                    |      |                      | |
| | Module Betazoid   | <-->  |                    | -->  | Exposes interface    | |
| +-------------------+       |                    |      | defined in ICS 25.   | |
|                             |                    |      |                      | |
| +-------------------+       |                    |      |                      | |
| | Module Cephalopod | <-->  |                    |      |                      | |
| +-------------------+       +--------------------+      +----------------------+ |
|                                                                                  |
+----------------------------------------------------------------------------------+
```

### Consensus state introspection

호스트 상태 머신은 `getCurrentHeight`를 사용하여 현재 높이를 검사 할 수있는 기능을 제공해야합니다.

```
type getCurrentHeight = () => uint64
```

호스트 상태 머신은 표준 이진 직렬화로 [ICS 2 ](../ics-002-client-semantics)의 요구 사항을 충족하는 고유한 ` ConsensusState` 유형을 정의해야합니다.

Host state machines MUST provide the ability to introspect their own consensus state, with `getConsensusState`:

```typescript
type getConsensusState = (height: uint64) => ConsensusState
```

`getConsensusState` MUST return the consensus state for at least some number `n` of contiguous recent heights, where `n` is constant for the host state machine. Heights older than `n` MAY be safely pruned (causing future calls to fail for those heights).

호스트 상태 머신은 `getStoredRecentConsensusStateCount` 하여 저장된 최근 컨센서스 상태 카운트 `n`을 조회 할 수 있는 기능을 제공해야합니다.

```typescript
type getStoredRecentConsensusStateCount = () => uint64
```

### Commitment Path Introspection

호스트 체인은 `getCommitmentPrefix`를 사용하여 commitment path를 검사 할 수있는 기능을 제공해야합니다.

```typescript
type getCommitmentPrefix = () => CommitmentPrefix
```

The result `CommitmentPrefix` is the prefix used by the host state machine's key-value store.
With the `CommitmentRoot root` and `CommitmentState state` of the host state machine, the following property MUST be preserved:

```typescript
if provableStore.get(path) === value {
  prefixedPath = applyPrefix(getCommitmentPrefix(), path)
  if value !== nil {
    proof = createMembershipProof(state, prefixedPath, value)
    assert(verifyMembership(root, proof, prefixedPath, value))
  } else {
    proof = createNonMembershipProof(state, prefixedPath)
    assert(verifyNonMembership(root, proof, prefixedPath))
  }
}
```

For a host state machine, the return value of `getCommitmentPrefix` MUST be constant.

### Port system

Host state machines MUST implement a port system, where the IBC handler can allow different modules in the host state machine to bind to uniquely named ports. Ports are identified by an `Identifier`.

Host state machines MUST permission interaction with the IBC handler such that:

- 모듈이 포트에 바인딩되면 모듈이 포트를 풀 때까지 다른 모듈은 해당 포트를 사용할 수 없습니다
- A single module can bind to multiple ports
- 포트는 선착순으로 할당되며 상태 머신이 처음 시작될 때 알려진 모듈에 대한 "예약 된" 포트를 바인딩 할 수 있습니다

This permissioning can be implemented with unique references (object capabilities) for each port (a la the Cosmos SDK), with source authentication (a la Ethereum), or with some other method of access control, in any case enforced by the host state machine. See [ICS 5](../ics-005-port-allocation) for details.

특정 IBC 기능을 사용하려는 모듈은 특정 핸들러 기능을 구현할 수 있습니다 (예: 다른 상태 머신의 관련 모듈을 사용하여 채널 핸드 셰이크에 로직 추가).

### Datagram submission

라우팅 모듈을 구현하는 호스트 상태 머신은 트랜잭션에 포함될 [ 데이터 그램 ](../../docs/ibc/2_IBC_TERMINOLOGY.md)을 ([ICS 26](../ics-026-routing-module)에 정의된) 제출하도록 `submitDatagram` 함수를 정의 할 수 있습니다.

```typescript
type submitDatagram = (datagram: Datagram) => void
```

`submitDatagram`을 사용하면 relayer process가 IBC 데이터 그램을 호스트 상태 머신의 라우팅 모듈에 직접 제출할 수 있습니다. 호스트 상태 머신은 데이터 그램을 제출하는 relayer process가 거래 수수료를 지불하고 더 큰 트랜잭션 구조에서 데이터 그램에 서명하는 계정을 가지고 있어야합니다. `submitDatagram`은 필요한 패키징을 정의하고 구성해야합니다.

### 예외 시스템

호스트 상태 머신은 예외 시스템을 지원해야하며, 이를 통해 트랜잭션은 실행을 중단할 수 있고 이전에 수행 한 상태 변경 (같은 트랜잭션 내에서 발생하는 다른 모듈의 상태 변경 포함 및 적절한 가스 소비 및 수수료 지불 포함)을 되돌릴 수 있습니다. 그리고 시스템 불변 위반은 상태 머신을 정지시킬 수 있습니다.

이 예외 시스템은 `abortTransactionUnless` 및 `abortSystemUnless` 두 함수를 통해 노출되어야 합니다. 전자는 트랜잭션을 revert 시키며 후자는 상태 머신을 정지시킵니다.

```typescript
type abortTransactionUnless = (bool) => void
```

`abortTransactionUnless`로 전달 된 bool 값이 `true`인 경우 호스트 상태 시스템은 아무 작업도 수행 할 필요가 없습니다. `abortTransactionUnless` 로 전달 된 bool 값이 `false`인 경우 호스트 상태 머신은 트랜잭션을 중단하고 가스 소비 및 수수료 지불을 제외하고 이전에 변경 한 상태 변경을 되돌려 야합니다.

```typescript
type abortSystemUnless = (bool) => void
```

`abortSystemUnless` 로 전달 된 bool 값이 `true` 인 경우 호스트 상태 시스템은 아무 작업도 수행 할 필요가 없습니다. `abortSystemUnless` 로 전달 된 bool 값이 `false` 인 경우 호스트 상태 시스템을 중지해야 합니다.

### 데이터 가용성

delivery-or-timeout safety를 위해 호스트 상태 머신은 최종 데이터 가용성을 가져야하며, 상태의 모든 key/value 쌍을 결국 relayer에 의해 검색 할 수 있습니다.  exactly-once safety를 위해 데이터 가용성이 필요하지 않습니다.

패킷 relay의 liveness을 위해 호스트 상태 머신은 transactional liveness을 제한해야하며 (따라서 반드시 consensus liveness를 가져야합니다.), 따라서 들어오는 트랜잭션은 블록 높이 제한 내에서 (특히, 패킷에 할당 된 시간 초과보다 작음) 반드시 confirm 됩니다.

IBC 패킷 데이터 및 상태 벡터에 직접 저장되지는 않지만 relayer에 의존하는 기타 데이터는 relayer process에 의해 효율적으로 계산 가능해야 합니다.

특정 합의 알고리즘의 라이트 클라이언트는 다르거나 더 엄격한 데이터 가용성 요구 사항을 가질 수 있습니다.

## 하휘 호환성

적용되지 않습니다.

## 상위 호환성

key/value store 기능 및 합의 상태(consensus state type) 유형은 단일 호스트 상태 시스템 작동 중에 변경되지 않을 수 있습니다

`submitDatagram`은 relayer가 프로세스를 업데이트 할 수 있어야 하므로 시간이 지남에 따라 변경 될 수 있습니다.

## 구현 예제

곧 게시될 예정입니다.

## 다른 구현

곧 게시될 예정입니다.

## 히스토리

29 April 2019 - Initial draft
11 May 2019 - Rename "RootOfTrust" to "ConsensusState"
25 June 2019 - Use "ports" instead of module names
18 August 2019 - Revisions to module system, definitions

## 저작권

이 게시물의 모든 내용은 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) 라이센스에 의해 보호받습니다.
