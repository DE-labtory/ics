---
ics: '20'
title: 대체 가능한(Fungible) 토큰 전송
stage: 초안
category: IBC/APP
requires: 25, 26
author: 'Christopher Goes '
created: '2019-07-15'
modified: '2019-08-25'
---

## 개요

이 표준 문서는 패킷 데이터 구조와 상태 머신의 처리 로직, 그리고 분리된 체인의 두 모듈 사이에서 IBC 채널을 통해 전송되는 대체 가능한(Fungible) 토큰을 위한 인코딩 방법에 대해 서술합니다. 제시된 상태 머신 로직은 허가되지 않은 채널 개방으로 부터 다중 체인 명령 처리를 안전하게 처리할 수 있습니다. 이 로직은 IBC 라우팅 모듈과 호스트 상태 머신의 자산 추적 모듈사이의 인터페이스를 정의하는 "대체 가능한 토큰을 전송하는 브릿지 모듈"을 구성합니다.

### 동기 (Motivation)

IBC 프로토콜로 연결된 한 그룹의 체인을 사용하는 유저들은 다른 체인에서 발행된 자산을 사용하길 원할 것이고, 교환이나 개인정보 보호와 같은 추가 기능을 사용하면서 발행 체인의 원래 자산과의 기능을 유지하고자 할 수 있습니다. 이 애플리케이션 레이어 표준은 IBC로 연결된 체인들 사이에서 자산의 대체 가능성과 소유권을 보호하고, 비잔틴 실패 시의 효과를 제한하며 추가적인 허가를 요구하지 않으면서 대체가능한 토큰을 전송하는 프로토콜에 대해 서술합니다.

### 정의

IBC 핸들러 인터페이스 & IBC 라우팅 모듈 인터페이스는 각각 [ICS 25](../ics-025-handler-interface)와 [ICS 26](../ics-026-routing-module)로 정의됩니다.

### 원하는 속성

- 대체가능성 보존 (Two-way peg).
- 총 공급량 보존 (단일 소스 체인 & 모듈에서 상수 또는 인플레이션)
- 허용 목록에 연결, 모듈, 명칭등을 추가할 필요 없는 무허가 토큰 이체
- 대칭성 (모든 체인은 같은 로직을 구현해야 하며, 허브 & 구역의 프로토콜 차별화가 없습니다.)
- 결함 억제: 체인 `B`의 비잔틴 행동(체인 `B`에 토큰을 보내는 유저들이 위험에 처할 수 있음)의 결과로 체인 `A`에서 발행된 토큰이나, 체인 `A` 자체에 발생할 수 있는 비잔틴 팽창

## 기술적 명세

### 자료 구조

오로지 명칭, 수량, 보내는 주소, 받는 주소, 보내는 체인이 자산의 소스인지 여부를 포함하는 패킷 데이터 타입인 `FungibleTokenPacketData`이 요구됩니다.

```typescript
interface FungibleTokenPacketData {
  denomination: string
  amount: uint256
  sender: string
  receiver: string
  source: boolean
}
```

대체가능한 토큰 전송 브릿지 모듈은 상태의 특정 채널과 관련된 에스크로 주소를 추적합니다. `ModuleState`의 필드들은 스코프 안에 있는 것으로 가정합니다.

```typescript
interface ModuleState {
  channelEscrowAddresses: Map<Identifier, string>
}
```

### 서브 프로토콜

여기서 설명하는 하위 프로토콜은 bank 모듈 및 IBC 라우팅 모듈에 대한 접근 권한이 있는 "대체 가능한 토큰 전송 브릿지" 모듈에서 구현되어야 합니다.

#### 포트 & 채널 설정

`setup` 함수는 적절한 포트에 바인드 되고, 에스크로 주소(모듈이 소유하는)를 생성하기 위해, 처음 모듈이 생성될 때(아마도 블록체인이 초기화 될 때) 한번만 호출되어야 합니다.

```typescript
function setup() {
  routingModule.bindPort("bank", ModuleCallbacks{
    onChanOpenInit,
    onChanOpenTry,
    onChanOpenAck,
    onChanOpenConfirm,
    onChanCloseInit,
    onChanCloseConfirm,
    onRecvPacket,
    onTimeoutPacket,
    onAcknowledgePacket,
    onTimeoutPacketClose
  })
}
```

`setup` 함수가 호출될 때, 채널은 분리된 체인의 대체가능한 토큰 전송 모듈 인스턴스 사이의 IBC 라우팅 모듈을 통해 생성될 수 있습니다.

관리자(호스트 상태 머신의 연결 & 채널을 만들 수 있는 권한이 있는)는 
 다른 상태 머신과 연결을 설정해야 하고, 다른 체인에 존재하는 이 모듈의 다른 인스턴스(또는 이 인터페이스를 돕는 다른 모듈)와의 채널을 생성해야 합니다. 이 규격은 패킷 처리의 시멘틱만을 정의하며, 모듈 자체가 어떤 연결이나 채널이 존재하는지에 대해 알 필요가 없는 방식으로 정의합니다.

#### 라우팅 모듈 콜백

##### 채널 생명주기 관리

`A` 머신과 `B` 머신 모두 다른 머신의 어떤 모듈로부터 새로운 채널 연결을 허용합니다.

- 다른 모듈은 "bank" 포트를 통해 바운드됩니다.
- 채널은 순서 없이 만들어집니다.
- 버전 문자열은 비어있습니다.

```typescript
function onChanOpenInit(
  order: ChannelOrder,
  connectionHops: [Identifier],
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  version: string) {
  // only unordered channels allowed
  abortTransactionUnless(order === UNORDERED)
  // only allow channels to "bank" port on counterparty chain
  abortTransactionUnless(counterpartyPortIdentifier === "bank")
  // version not used at present
  abortTransactionUnless(version === "")
  // allocate an escrow address
  channelEscrowAddresses[channelIdentifier] = newAddress()
}
```

```typescript
function onChanOpenTry(
  order: ChannelOrder,
  connectionHops: [Identifier],
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  version: string,
  counterpartyVersion: string) {
  // only unordered channels allowed
  abortTransactionUnless(order === UNORDERED)
  // version not used at present
  abortTransactionUnless(version === "")
  abortTransactionUnless(counterpartyVersion === "")
  // only allow channels to "bank" port on counterparty chain
  abortTransactionUnless(counterpartyPortIdentifier === "bank")
  // allocate an escrow address
  channelEscrowAddresses[channelIdentifier] = newAddress()
}
```

```typescript
function onChanOpenAck(
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  version: string) {
  // version not used at present
  abortTransactionUnless(version === "")
  // port has already been validated
}
```

```typescript
function onChanOpenConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
  // accept channel confirmations, port has already been validated
}
```

```typescript
function onChanCloseInit(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
  // no action necessary
}
```

```typescript
function onChanCloseConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
  // no action necessary
}
```

##### 패킷 릴레이

평문으로, 체인 `A`와 `B` 사이에서:

- 소스 영역에서, 브릿지 모듈은 송신 체인의 기존 지역 자산 분포와 수신 체인의 민트 증권을 은닉합니다.
- Sink 영역에서 브릿지 모듈은 송신 체인의 지역 보증을 소각하고, 수신 체인에서의 지역 자산 디노미네이션을 해제합니다.
- 패킷이 시간 내에 도착하지 못한 경우, 로컬 자산은 송신자에게 다시 예치되지 않거나, 적절히 다시 돌아갑니다.
- 승인 데이터는 필요하지 않습니다.

`createOutgoingPacket`는 반드시 호스트 상태 머신의 계정 소유자에게 특정하여 적절한 서명 여부를 체크하는 트랜잭션 핸들러에 의해 호출되어야 합니다.

```typescript
function createOutgoingPacket(
  denomination: string,
  amount: uint256,
  sender: string,
  receiver: string,
  source: boolean) {
  if source {
    // sender is source chain: escrow tokens
    // determine escrow account
    escrowAccount = channelEscrowAddresses[packet.sourceChannel]
    // construct receiving denomination, check correctness
    prefix = "{packet/destPort}/{packet.destChannel}"
    abortTransactionUnless(denomination.slice(0, len(prefix)) === prefix)
    // escrow source tokens (assumed to fail if balance insufficient)
    bank.TransferCoins(sender, escrowAccount, denomination.slice(len(prefix)), amount)
  } else {
    // receiver is source chain, burn vouchers
    // construct receiving denomination, check correctness
    prefix = "{packet/sourcePort}/{packet.sourceChannel}"
    abortTransactionUnless(denomination.slice(0, len(prefix)) === prefix)
    // burn vouchers (assumed to fail if balance insufficient)
    bank.BurnCoins(sender, denomination, amount)
  }
  FungibleTokenPacketData data = FungibleTokenPacketData{denomination, amount, sender, receiver, source}
  handler.sendPacket(packet)
}
```

`onRecvPacket`는 전달되는 패킷이 수신되면, 라우팅 모듈에 의해 호출됩니다.

```typescript
function onRecvPacket(packet: Packet): bytes {
  FungibleTokenPacketData data = packet.data
  if data.source {
    // sender was source chain: mint vouchers
    // construct receiving denomination, check correctness
    prefix = "{packet/destPort}/{packet.destChannel}"
    abortTransactionUnless(data.denomination.slice(0, len(prefix)) === prefix)
    // mint vouchers to receiver (assumed to fail if balance insufficient)
    bank.MintCoins(data.receiver, data.denomination, data.amount)
  } else {
    // receiver is source chain: unescrow tokens
    // determine escrow account
    escrowAccount = channelEscrowAddresses[packet.destChannel]
    // construct receiving denomination, check correctness
    prefix = "{packet/sourcePort}/{packet.sourceChannel}"
    abortTransactionUnless(data.denomination.slice(0, len(prefix)) === prefix)
    // unescrow tokens to receiver (assumed to fail if balance insufficient)
    bank.TransferCoins(escrowAccount, data.receiver, data.denomination.slice(len(prefix)), data.amount)
  }
  return 0x
}
```

`onAcknowledgePacket`는 전송한 패킷이 승인되었을때 라우팅 모듈에 의해 호출됩니다.

```typescript
function onAcknowledgePacket(
  packet: Packet,
  acknowledgement: bytes) {
  // nothing is necessary, likely this will never be called since it's a no-op
}
```

`onTimeoutPacket`는 수신한 패킷이 타임아웃일때 실행됩니다.(이 패킷은 목적지 체인에 받아들여지지 않습니다.)

```typescript
function onTimeoutPacket(packet: Packet) {
  FungibleTokenPacketData data = packet.data
  if data.source {
    // sender was source chain, unescrow tokens
    // determine escrow account
    escrowAccount = channelEscrowAddresses[packet.destChannel]
    // construct receiving denomination, check correctness
    prefix = "{packet/sourcePort}/{packet.sourceChannel}"
    abortTransactionUnless(data.denomination.slice(0, len(prefix)) === prefix)
    // unescrow tokens back to sender
    bank.TransferCoins(escrowAccount, data.sender, data.denomination.slice(len(prefix)), data.amount)
  } else {
    // receiver was source chain, mint vouchers
    // construct receiving denomination, check correctness
    prefix = "{packet/sourcePort}/{packet.sourceChannel}"
    abortTransactionUnless(data.denomination.slice(0, len(prefix)) === prefix)
    // mint vouchers back to sender
    bank.MintCoins(data.sender, data.denomination, data.amount)
  }
}
```

```typescript
function onTimeoutPacketClose(packet: Packet) {
  // can't happen, only unordered channels allowed
}
```

#### 추론

##### 정확성

이 구현은 대체가능성과 공급을 모두 보전합니다.

대체가능성: 토큰이 상대방 체인으로 전송될 때, 소스 체인의 동일한 금액으로 상환될 수 있습니다.

공급: 공급을 토큰의 잠금을 해제함으로써 재정의합니다. 모든 송-수신 쌍의 합은 0입니다. 소스 체인은 공급을 유동적으로 조절합니다.

##### 멀티 체인 노트

이것은 아직 유저가 체인 A에서 만들어진 토큰을 체인 B로 보내고, 다시 채널 D로 보낸 후, 그것을 다시 채널 D -> C -> A를 통해 되돌려 받는 "다이아몬드 문제"에 대해 다루지 않습니다 — 왜냐하면 공급은 체인 B가 소유한 것으로 추적되고, 체인 C는 중개 역할을 할 수 없기 때문입니다. 이것을 프로토콜 내에서 처리해야 할 지는 아직 분명하지 않습니다. 단지 원래 상환 경로만 요구하는 것이 더 나을수도 있습니다.(그리고 만약 유동성이 많고 양쪽 경로에 약간의 잉여분이 있다면, 다이아몬드 경로는 효과가 있을 것입니다.) 긴 상환 경로에서 발생하는 복잡성은 네트워크 도폴로지에서 중심 사슬의 출현으로 이어질 수 있습니다.

#### 선택 부록

- 각 체인은 로컬에서 패킷을 주고받을 때 더 긴 단위로 번역되는 짧고 사용자 친화적인 지역 단위를 사용하기 위한 lookup 테이블을 선택할 수 있습니다.
- 다른 머신들과 연결될 수 있는 채널과 설정될 수 있는 채널에 추가적인 제한이 부과될 수 있습니다.

## 백워드 호환성

적용되지 않습니다.

## 포워드 호환성

앞으로 나올 이 표준의 버전은 다른 버전과 채널을 생성할 수 있습니다.

## 구현 예제

곧 게시될 예정입니다.

## 다른 구현

곧 게시될 예정입니다.

## 히스토리

2019년 7월 15일 - 초안 작성
2019년 7월 29일 - 주요 수정; 정리
2019년 8월 25일 - 주요 수정; 더 많은 정리

## Copyright

이 게시물의 모든 내용은 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) 라이센스에 의해 보호받습니다.
