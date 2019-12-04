---
ics: '5'
title: 포트 할당
stage: 초안
requires: '24'
required-by: '4'
category: IBC/TAO
author: Christopher Goes <cwgoes@tendermint.com>
created: '2019-06-20'
modified: '2019-08-25'
---

## 개요

이 표준은 IBC 핸들러에 의해 할당된 고유한 이름의 포트에 모듈을 바인딩 할 수 있는 포트 할당 시스템에 관한 설명입니니다. 이렇게 할당된 포트들은 채널을 여는데 사용될 수 있으며, 바인딩된 모듈에 의해 포트는 소유권이 이전되거나 나중에 해제될 수 있습니다.

### 의도

블록체인 간 통신 프로토콜은 모듈 간 트래픽을 촉진하도록 설계되었습니다. 여기서 모듈은 독립적이며 상호 신뢰할 수 없으며, 주권 원장(sovereign ledger)에서 실행되는 독립적인 코드 요소입니다. IBC에서 요구되는 종단간 시멘틱(semantics)을 제공하려면 IBC 핸들러가 특정 모듈에 대한 채널을 허용해야합니다.
이 사양은 해당 모델을 실행하는 *포트 할당 및 소유권* 시스템을 정의합니다.

대체 가능한(fungible) 토큰 처리를 위한 "뱅크" 또는 체인 간 담보를 위한 "스테이킹"과 같이 특정 포트 이름에 어떤 종류의 모듈 로직이 바인딩되어 있는지에 대한 규칙이 나타날 수 있습니다.
이는 HTTP 서버에서 포트 80의 일반적인 사용과 유사합니다. - 프로토콜은 특정 모듈 로직이 실제로 기존 포트에 바인딩 되도록 강제할 수 없으므로 사용자 스스로 확인해야합니다. 임시 프로토콜 처리를 위해 의사 난수 식별자가 있는 임시 포트를 만들 수 있습니다.

모듈은 여러 개의 포트에 바인딩될 수 있으며 별도의 머신에 존재하는 다른 모듈에 의해 바인딩된 여러 개의 포트에 연결될 수 있습니다. 여러 (고유하게 식별된) 채널들이 동시에 하나의 포트를 사용할 수 있습니다. 채널은 두 포트 사이에서 종단 간이며, 각각의 포트는 모듈에 의해 먼저 바인딩되어 있어야 하며, 채널의 마지막을 제어합니다.

호스트 상태 머신은 포트에 바인딩할 수 있는 기능 키(capability key)를 생성함으로써, 특별한 권한이 부여된 모듈 관리자에게만 포트 바인딩을 선택적으로 노출하도록 할 수 있습니다. 그런 다음 모듈 관리자는 사용자 정의 규칙에 의해 어떤 포트가 어떤 모듈로 바인딩 할 수 있는지를 제어할 수 있고, 포트 이름 및 모듈의 유효성을 검사가 된 경우에만 포트를 모듈로 전송할 수 있습니다. 이 역할은 라우팅 모듈에서 수행할 수 있습니다 ( [ICS 26](../ics-026-routing-module) 참조).

### 정의

`Identifier` , `get` , `set` 및 `delete`는 [ICS 24](../ics-024-host-requirements)에서 정의하고 있습니다.

*포트*는 채널 열기 및 모듈 사용을 허용하는데 사용되는 특별한 종류의 식별자입니다.

*모듈*은 IBC 핸들러와 무관한 호스트 상태 머신의 하위 구성 요소입니다. 모듈의 예로는 Ethereum 스마트 컨트랙트 및 Cosmos SDK & Substrate 모듈 등이 있습니다. IBC 사양은 호스트 상태 머신이 오브젝트 기능(object-capability) 또는 출발지 인증(source authentication)을 사용하여 모듈에 포트에 대한 권한을 부여하는 것 이외의 모듈 기능을 가정하지 않습니다.

### 지향 속성

- 일단 모듈이 포트에 바인딩되면 해당 모듈이 스스로 해제하기 전까지 다른 모듈들은 해당 포트를 사용할 수 없습니다.
- 모듈은 옵션에 따라 포트를 해제하거나 다른 모듈로 소유권을 이전할 수 있습니다.
- 하나의 모듈은 여러 포트에 한 번에 바인딩될 수 있습니다.
- 포트들은 선착순으로 할당되며, 알려진 모듈에 대한 "예약된" 포트들은 체인이 처음 시작될 때 바인딩될 수 있습니다.

유용한 비교로서, 다음과 같은 TCP와의 비교가 있으며 이는 대략 정확합니다:

IBC 컨셉 | TCP/IP 컨셉 | 차이점
--- | --- | ---
IBC | TCP | 많음, IBC를 설명하는 아키텍처 문서를 참조하세요
포트 (예시. "은행") | 포트 (예시. 80) | TCP는 낮은 수의 숫자들인 반면, IBC는 문자열로 포트를 식별합니다
모듈 (예시. "은행") | 응용 프로그램 (예시. Nginx) | 응용 프로그램 별로 다릅니다
클라이언트 | - | 직접적인 비교할 수 있는 부분 없음, L2 라우팅 또는 TLS와 유사합니다
커넥션 | - | 직접적인 비교할 수 있는 부분 없음, TCP의 Connection들에 포함됩니다.
채널 | 커넥션 | 여러 채널을 동시에 포트로부터 열거나 포트로 열 수 있습니다

## 기술 사양

### 자료 구조

호스트 상태 머신은 모듈에 대한 (1) 오브젝트 기능(object-capability) 또는 (2) 출발지 인증(source authentication)을 지원해야 합니다.

(1) 오브젝트 기능(object-capability)의 경우, IBC 핸들러는 *오브젝트 기능(object-capability)*과 모듈로 전달 될 수 있으며 다른 모듈에서는 복제 할 수 없는 고유하고 불투명한 참조를 생성할 수 있어야 합니다. 각각의 예는 Cosmos SDK에서 사용되는 상점 키 ( [참조](https://github.com/cosmos/cosmos-sdk/blob/master/store/types/store.go#L224) )와 Agoric의 Javascript 런타임 ( [reference](https://github.com/Agoric/SwingSet) )에서 사용되는 객체 참조입니다.

```typescript
type CapabilityKey object
```

```typescript
function newCapabilityPath(): CapabilityKey {
  // provided by host state machine, e.g. pointer address in Cosmos SDK
}
```

(2) 출발지 인증(source authentication)의 경우, IBC 핸들러는 호출하는 모듈의 *출발지 식별자(source identifier)*와 모듈에 의해 변경되거나 다른 모듈에 의해 위조 될 수 없는 호스트 상태 머신의 각 모듈에 대한 고유한 문자열을 안전하게 읽을 수 있어야합니다. 예시로는 Ethereum에서 사용되는 스마트 컨트랙트 주소 ( [reference](https://ethereum.github.io/yellowpaper/paper.pdf) )가 있습니다.

```typescript
type SourceIdentifier string
```

```typescript
function callingModuleIdentifier(): SourceIdentifier {
  // provided by host state machine, e.g. contract address in Ethereum
}
```

`generate` 및 `authenticate` 함수는 다음과 같이 정의됩니다.

(1) 오브젝트 기능(object-capability)의 경우, `generate` 함수는 외부 계층 함수에 의해 반환되어야 하는 새로운 오브젝트 기능(object-capability)키를 리턴해야 하며, 그리고 `authenticate` 함수는 외부 계층 함수에서 호스트 상태 머신에 의해 고유성이 강화된 오브젝트 기능(object-capability)키인 `capability`를 추가 인자로 받을 것을 요구합니다. 외부 계층 함수들은 IBC 핸들러 ( [ICS 25](../ics-025-handler-interface) ) 또는 라우팅 모듈 ( [ICS 26](../ics-026-routing-module) )에 의해 모듈로 노출되는 함수들입니다.

```
function generate(): CapabilityKey {
    return newCapabilityPath()
}
```

```
function authenticate(key: CapabilityKey): boolean {
    return capability === key
}
```

(2) 출발지 인증(source authentication)의 경우, `generate` 함수는 호출 모듈의 식별자를 반환하고 `authenticate` 함수는 단순히 그것을 확인합니다.

```
function generate(): SourceIdentifier {
    return callingModuleIdentifier()
}
```

```
function authenticate(id: SourceIdentifier): boolean {
    return callingModuleIdentifier() === id
}
```

#### 저장 경로

`portPath` 는 `Identifier` 를 가져 와서 포트와 연관된 오브젝트 기능(object-capability) 참조 또는 오너 모듈 식별자가 저장될 저장소 경로를 반환합니다.

```typescript
function portPath(id: Identifier): Path {
    return "ports/{id}"
}
```

### 하위 프로토콜

#### 식별자 검증

포트의 소유자 모듈 식별자는 고유한 `Identifier` 접두사 아래 저장됩니다.
검증 함수인 `validatePortIdentifier` 가 제공될 수 있습니다.

```typescript
type validatePortIdentifier = (id: Identifier) => boolean
```

검증 함수가 제공되지 않으면, 기본적으로 `validatePortIdentifier` 함수가 항상 `true`를 반환합니다.

#### 포트 바인딩

IBC 핸들러는 반드시 `bindPort` 구현하고 있어야 합니다. `bindPort` 는 할당되지 않은 포트를 찾아 바인딩하며, 포트가 이미 할당되어 있는 경우에는 실패하게 됩니다.

호스트 상태 머신이 포트 할당을 제어하기 위해 특수 모듈 관리자를 구현하지 않으면 모든 모듈에서 `bindPort` 사용할 수 있어야 합니다. 이 경우, `bindPort` 는 모듈 관리자만 호출할 수 있어야 합니다.

```typescript
function bindPort(id: Identifier) {
    abortTransactionUnless(validatePortIdentifier(id))
    abortTransactionUnless(privateStore.get(portPath(id)) === null)
    key = generate()
    privateStore.set(portPath(id), key)
    return key
}
```

#### 포트 소유권 이전

호스트 상태 머신이 객체 기능(object-capabilities)을 지원하는 경우, 포트 참조가 베어러 기능(bearer capability)이므로 추가 프로토콜이 필요하지 않습니다. 그렇지 않은 경우 IBC 핸들러는 다음 `transferPort` 함수를 구현할 수 있습니다.

모든 모듈에서 `transferPort`를 사용할 수 있어야 합니다.

```typescript
function transferPort(id: Identifier) {
    abortTransactionUnless(authenticate(privateStore.get(portPath(id))))
    key = generate()
    privateStore.set(portPath(id), key)
}
```

#### 포트 해제

IBC 핸들러는 `releasePort` 함수를 구현해야하며, 이를 통해 모듈은 다른 모듈이 해당 포트에 바인딩될 수 있도록 포트를 해제할 수 있습니다.

모든 모듈에서 `releasePort`는 사용할 수 있어야 합니다.

> 경고 : 포트를 해제하면 다른 모듈이 해당 포트에 바인딩되어 들어오는 채널 오프닝 핸드셰이크를 가로챌 수 있습니다. 모듈은 안전한 경우에만 포트를 해제해야 합니다.

```typescript
function releasePort(id: Identifier) {
    abortTransactionUnless(authenticate(privateStore.get(portPath(id))))
    privateStore.delete(portPath(id))
}
```

### 속성 및 불변량

- 기본적으로 포트 식별자는 선착순입니다. 일단 모듈이 포트에 바인딩되면 모듈이 포트의 소유권을 이전하거나 해제할 때까지 해당 모듈만 포트를 활용할 수 있습니다. 모듈 관리자는 이를 재정의(override)하는 커스텀 로직을 구현할 수 있습니다.

## 하위 호환성

적용되지 않습니다.

## 상위 호환성

포트 바인딩은 유선 프로토콜이 아니므로 소유권 의미(semantics)가 영향을 받지 않는 한 인터페이스가 별도의 체인에서 독립적으로 변경 될 수 있습니다.

## 구현 예제

곧 게시될 예정입니다.

## 다른 구현

곧 게시될 예정입니다.

## 히스토리

2019년 6월 29일 - 초안

## 저작권

이 게시물의 모든 내용은 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) 라이센스에 의해 보호받습니다.
