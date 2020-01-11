---
ics: '20'
title: 대체 가능한(Fungible) 토큰 전송
stage: 초안
category: IBC/APP
requires: 25, 26
kind: instantiation
author: Christopher Goes <cwgoes@tendermint.com>
created: '2019-07-15'
modified: '2019-08-25'
---

## 개요

이 표준 문서는 패킷 데이터 구조와 상태 머신의 처리 로직, 그리고 분리된 체인의 두 모듈 사이에서 IBC 채널을 통한 대체 가능한(Fungible) 토큰의 전송을 위한 인코딩 방법에 대해 서술합니다. 제시된 상태 머신 로직은 허가하지 않은 체인 개방에서도 다중 체인의 토큰 명칭을 안전하게 처리하도록 고려합니다. 이 로직은 IBC 라우팅 모듈과 호스트 상태 머신의 자산 추적 모듈 사이의 인터페이스를 정의하는 "대체 가능한 토큰을 전송하는 브릿지 모듈"을 구성합니다.

### 동기

IBC 프로토콜로 연결된 체인 그룹의 유저들은 한 체인에서 발행된 자산을 다른 체인에서도 사용하고, 아마 교환이나 개인정보 보호와 같은 추가 기능을 사용하면서 발행 체인에서 본래 자산이 가진 기능을 유지하길 원할 것 입니다. 이 애플리케이션 계층 표준은 IBC로 연결된 체인들 사이에서 대체 가능한 토큰을 전송하는 프로토콜에 대해 서술합니다. IBC는 자산의 대체 가능성과 소유권을 보존하고, 비잔틴 실패의 영향를 제한하며, 추가적인 권한을 필요로 하지 않습니다.

### 정의

IBC 핸들러 인터페이스와 IBC 라우팅 모듈 인터페이스는 각각 [ICS 25](../ics-025-handler-interface)와 [ICS 26](../ics-026-routing-module)에서 정의됩니다.

### 원하는 속성

- 대체가능성 보존 (Two-way peg).
- 총 공급량 보존 (단일 소스 체인과 모듈에서 상수이거나 인플레이션)
- 연결, 모듈, 자산 명칭의 허용 목록이 필요 없는, 권한이 필요 없는 토큰 이체
- 대칭성 (모든 체인은 같은 로직을 구현하며, 허브(Hub)와 존(Zone)의 프로토콜 차이가 없음)
- 결함 방지 : 체인 `B`의 비잔틴 행동으로 인한, 체인 `A`에서 발생하는 토큰의 비잔틴 인플레이션을 방지 (체인 `B`로 토큰을 보낸 사용자가 위험할 수 있더라도)

## 기술적 명세

### 자료 구조

화폐 명칭, 수량, 보내는 주소, 받는 주소, 보내는 체인이 자산의 근원인지 여부를 특정하는 패킷 데이터 타입 `FungibleTokenPacketData`만이 요구됩니다.

```typescript
interface FungibleTokenPacketData {
  denomination: string
  amount: uint256
  sender: string
  receiver: string
  source: boolean
}
```

대체 가능한 토큰 전송 브릿지 모듈은 상태의 특정 채널과 관련된 에스크로 주소를 추적합니다. `ModuleState`의 필드는 범위 안에 있다고 가정합니다.

```typescript
interface ModuleState {
  channelEscrowAddresses: Map<Identifier, string>
}
```

### 하위 프로토콜

여기서 설명하는 하위 프로토콜은 bank 모듈 및 IBC 라우팅 모듈에 대한 접근 권한이 있는 "대체 가능한 토큰 전송 브릿지" 모듈에서 구현되어야 합니다.

#### 포트 & 채널 설정

`setup` 함수는 적절한 포트에 바인드하고, (모듈이 소유하는) 에스크로 주소를 생성하기 위해, 처음 모듈이 생성될 때 (아마도 블록체인이 초기화될 때) 한번만 호출되어야 합니다.

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

`setup` 함수가 호출될 때, 별개의 체인에 있는 대체 가능한 토큰 전송 모듈 인스턴스 사이의 IBC 라우팅 모듈을 통해 채널을 생성할 수 있습니다.

(호스트 상태 머신을 연결하고 채널을 만들 수 있는 권한이 있는) 관리자는 다른 상태 머신과 연결을 설정하고, 다른 체인에 존재하는 이 모듈의 다른 인스턴스(또는 이 인터페이스를 돕는 다른 모듈)와의 채널을 생성해야 합니다. 이 규격은 패킷 처리 시멘틱만을 정의하며, 모듈 자체가 어떤 연결이나 채널이 특정 시간에 존재하는지에 대해 알 필요가 없도록 정의합니다.

#### 라우팅 모듈 callback

##### 채널 생명주기 관리

`A` 머신과 `B` 머신은 둘 다 다음이 성립해야만 다른 머신의 모든 모듈로부터 새로운 채널 연결을 허용합니다.

- 다른 모듈은 "bank" 포트에 바운드합니다.
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

##### 패킷 전달

쉽게 말하여, 체인 `A`와 `B` 사이에서:

- 출발지 존(zone)에서, 브릿지 모듈은 보내는 체인에 존재하는 로컬 자산을 조건부로 묶고, 받는 체인에 증권(voucher)을 발행합니다.
- 목적지 존(zone)에서, 브릿지 모듈은 보내는 체인의 로컬 증권(voucher)을 소각하고, 받는 체인에서 로컬 자산을 다시 돌려줍니다.
- 패킷이 시간 내에 도착하지 못한 경우, 로컬 자산은 보낸 사람에게 다시 환불되거나, 적절히 증권(voucher)로 발행됩니다.
- acknowledgement 정보는 필요하지 않습니다.

호스트 상태 머신에 있는 계정 소유자의 서명을 확인하는 모듈의 트랜잭션 핸들러는 반드시 `createOutgoingPacket`를 호출해야 합니다.

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

라우팅 모듈은 이 모듈로 전달되는 패킷을 수신하면, `onRecvPacket`를 호출합니다.

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

라우팅 모듈은 이 모듈이 전송한 패킷이 확인되었을 때 `onAcknowledgePacket`를 호출합니다.

```typescript
function onAcknowledgePacket(
  packet: Packet,
  acknowledgement: bytes) {
  // nothing is necessary, likely this will never be called since it's a no-op
}
```

라우팅 모듈은 이 모듈이 보낸 패킷이 (목적지 체인에 수신되지 않아서) 시간 초과했을 때, `onTimeoutPacket`를 호출합니다.

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

대체가능성: 토큰이 상대방 체인으로 전송된 경우, 이 토큰은 소스 체인에서 동일한 화폐 단위와 양으로 상환될 수 있습니다.

공급: 공급을 잠금이 해제된 토큰으로 재정의합니다. 모든 송수신 쌍의 합은 총 0입니다. 소스 체인은 공급량을 조절할 수 있습니다.

##### 멀티 체인 관련 내용

여기서는 아직 "다이아몬드 문제"를 다루지 않습니다. 다이아몬드 문제란, 사용자가 체인 A에서 만들어진 토큰을 체인 B로 보내고, 다시 체인 D로 보낸 다음, 체인 D에서 체인 C를 거쳐 체인 A로 돌려받는 경우를 말합니다. 이 문제를 다루지 않는 이유는, 체인 B가 공급을 소유한 것으로 추적되어, 체인 C가 중계 역할을 할 수 없기 때문입니다. 이런 경우가 프로토콜 안에서 처리되어야 할지 여부는 아직 분명하지 않습니다. 다만 본래의 상환 경로만을 사용하면 괜찮을 것입니다. (그리고 유동성이 많고 양쪽 경로에 약간의 잉여분이 있다면, 다이아몬드 경로는 대부분 작동합니다.) 긴 상환 경로에서 발생하는 복잡성 때문에 네트워크 토폴로지에서 중앙 체인이 나타날 수 있습니다.

#### 선택 부록

- 각 체인은 로컬에서 lookup 테이블을 가질 수 있습니다. 이 테이블은 패킷을 주고 받을 때 긴 명칭 대신 더 짧고 사용자 친화적인 로컬 명칭을 사용하도록 합니다.
- 어떤 다른 머신과 연결할지와 어떤 채널을 만들지에 대하여 추가적인 제한 사항이 있을 수 있습니다.

## 하위 호환성

적용되지 않습니다.

## 상위 호환성

앞으로 나올 이 표준의 버전은 채널 생성에 있어 다른 버전을 사용할 수 있습니다.

## 구현 예제

곧 게시될 예정입니다.

## 다른 구현

곧 게시될 예정입니다.

## 히스토리

2019년 7월 15일 - 초안 작성

2019년 7월 29일 - 주요 수정; 정리

2019 8월 25일 - 주요 수정 및 더 많은 정리

## 저작권

이 게시물의 모든 내용은 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) 라이센스에 의해 보호받습니다.
