---
ics: '23'
title: Vector Commitments
stage: 초안
required-by: 2, 24
category: IBC/TAO
author: 'Christopher Goes '
created: '2019-04-16'
modified: '2019-08-25'
---

## 개요

*Vector commitment*은 벡터의 색인 및 요소에 대해 색인화 된 벡터 및 짧은 멤버쉽 및 / 또는 비회원 증명에 대한 일정한 크기의 구속력있는 Commitment을 생성하는 구성입니다.
이 사양은 IBC 프로토콜에 사용 된 Commitment 구성에 필요한 기능과 특성을 열거합니다. 특히, IBC에서 활용되는 약속은 *위치적으로 구속력* 이 있어야합니다: 반드시 특정 위치에서 존재 혹은 비존재를 증명할 수 있어야 합니다.

### 동기 (Motivation)

다른 체인이 검증할 수 있는, 어떤 체인의 특정 상태(state) 전이를 보증하기 위해, IBC는 상태(state)의 특정 경로에 어떤 값에 대한 포함 혹은 비포함 여부를 증명할 수 있는 효율적인 암호화 구성을 요구합니다.

### 정의 (Definitions)

Vector commitment의 *관리자(manager)*는 Commitment의 아이템을 추가하거나 삭제할 수 있는 행위자 입니다. 일반적으로 블록체인의 상태 머신입니다.

*증명자(prover)*는 특정 원소의 포함 여부에 대한 증명을 제공합니다. 일반적으로 전이자(relayer) 입니다.([ICS 18](../ics-018-relayer-algorithms)를 참조하세요.)

*검증자(verifier)*는 commitment의 관리자가 특정 원소를 추가했는지 확인하기 위해 증명을 확인합니다. 일반적으로 다른 체인에서 동작하는 IBC 핸들러(IBC 모듈을 구현하는) 입니다.

Commitment들은 직렬화(Serialisable) 가능한 특정 *경로(path)*와 *값(value)*의 타입과 함께 객체화됩니다.

*무시가능한 함수(negligible function)*는 모든 양의 다항식의 역수보다 천천히 늘어나는 함수이고, [여기](https://en.wikipedia.org/wiki/Negligible_function) 정의되어 있습니다.

### 원하는 속성 (Desired Properties)

이 문서는 구체적인 구현이 아닌 원하는 속성에 대해서만 정의합니다. 구체적은 구현은 아래 "속성(Properties)" 을 참조하세요.

## 기술 명세 (Technical Specification)

### 데이터 타입 (Datatypes)

Commitment 구성은 반드시 다음 데이터 타입을 지정해야 합니다. 그렇지 않으면, 불투명하지만 (검사할 필요는 없지만) 직렬화가 가능해야 합니다.

#### Commitment State

`CommitmentState`는 관리자(manager)에 의해 저장되는 commitment의 전체 상태(state)입니다.

```typescript
type CommitmentState = object
```

#### Commitment Root

`CommitmentRoot`는 특정 commitment 상태를 커밋하며, 일정한 크기여야 합니다.

일정한 크기의 상태를 갖는 특정 commitment 구조에서, `CommitmentState`와 `CommitmentRoot`는 같은 타입일 것입니다.

```typescript
type CommitmentRoot = object
```

#### Commitment Path

`CommitmentPath`는 임의의 구조화된 객체(commitment type으로 정의된)의 commitment의 증명을 위해 사용되는 경로입니다. 이것은 반드시 `applyPrefix`에(아래 정의되어 있습니다.) 의해 계산되어야 합니다.

```typescript
type CommitmentPath = object
```

#### Prefix

`CommitmentPrefix`는 commitment 증명의 prefix 저장을 정의합니다. 이것은 검증 증명 함수가 경로를 통과하기 전에 적용됩니다.

```typescript
type CommitmentPrefix = object
```

`applyPrefix`는 매개변수로 부터 새로운 commitment 경로를 구성합니다. 이것은 prefix 매개변수의 문맥에 있는 경로 매개변수를 해석합니다.

`(prefix, path)` 튜플에 대해, `applyPrefix(prefix, path)`는 튜플의 원소가 같은 경우에, 반드시 같은 키를 반환해야 합니다.

`applyPrefix`는 반드시 각 `Path`에 대해 정의되어야 하며, `Path`는 각자 다른 구체적인 구조를 가질 수 있습니다. `applyPrefix`는 아마도 여러개의 `CommitmentPrefix` 타입들을 수용할 수 있습니다.

`applyPrefix`에 의해 반환된 `CommitmentPath`는 직렬화 가능할(serialisable) 필요는 없습니다. (예를 들어, 이것은 트리 노드 식별자의 리스트일 수 있습니다.) 그러나 이것은 서로 같음을 비교해야 합니다.

```typescript
type applyPrefix = (prefix: CommitmentPrefix, path: Path) => CommitmentPath
```

#### Proof

`CommitmentProof`는 알려진 commitment root와 함께 검증 가능한 한 원소나 원소 그룹의 멤버십 여부를 나타냅니다. 검증은 간결해야 합니다.

```typescript
type CommitmentProof = object
```

### Required functions

Commitment 구성은 반드시 경로를 통해 직렬화 가능한 오브젝트와, 바이트 배열형태로 정의한 다음 함수들을 제공해야 합니다,

```typescript
type Path = string

type Value = string
```

#### 초기화 (Initialisation)

`generate` 함수는 초기의 (아마도 비어있는)경로와 값을 대응하는 맵으로부터 commitment의 상태를 초기화합니다.

```typescript
type generate = (initial: Map<Path, Value>) => CommitmentState
```

#### 루트 계산 (Root calculation)

`calculateRoot` 함수는 증명을 위해 사용될 수 있는 commitment state에 대한 일정한 크기의 commitment를 계산합니다.

```typescript
type calculateRoot = (state: CommitmentState) => CommitmentRoot
```

#### 원소의 추가와 삭제 (Adding & removing elements)

`set` 함수는 commitment에 있는 값에 대한 경로를 설정합니다.

```typescript
type set = (state: CommitmentState, path: Path, value: Value) => CommitmentState
```

`remove` 함수는 경로와 연관된 commitment의 값을 함께 제거합니다.

```typescript
type remove = (state: CommitmentState, path: Path) => CommitmentState
```

#### 증명 생성 (Proof generation)

`createMembershipProof` 함수는 특정한 commitment 경로가 commitment의 어떤 값으로 보내졌다는 증명을 만들어냅니다.

```typescript
type createMembershipProof = (state: CommitmentState, path: CommitmentPath, value: Value) => CommitmentProof
```

`createNonMembershipProof` 함수는 commitment 경로가 commitment의 어떠한 값으로도 보내지지 않았다는 증명을 만들어냅니다.

```typescript
type createNonMembershipProof = (state: CommitmentState, path: CommitmentPath) => CommitmentProof
```

#### 증명 검증 (Proof verification)

`verifyMembership` 함수는 어떤 commitment의 값이 경로로 보내졌다는 증명을 검증합니다.

```typescript
type verifyMembership = (root: CommitmentRoot, proof: CommitmentProof, path: CommitmentPath, value: Value) => boolean
```

`verifyNonMembership` 함수는 어떤 commitment의 값이 경로로 보내지지 않았다는 증명을 검증합니다.

```typescript
type verifyNonMembership = (root: CommitmentRoot, proof: CommitmentProof, path: CommitmentPath) => boolean
```

### Optional functions

Commitment 구축은 다음의 함수를 제공할 수 있습니다.

`batchVerifyMembership` 함수는 commitment의 값이 많은 경로로 보내졌다는 증명을 검증합니다.

```typescript
type batchVerifyMembership = (root: CommitmentRoot, proof: CommitmentProof, items: Map<CommitmentPath, Value>) => boolean
```

`batchVerifyNonMembership` 함수는 commitment의 어떤 값도 많은 경로로 보내지지 않았았다는 증명을 검증합니다.

```typescript
type batchVerifyNonMembership = (root: CommitmentRoot, proof: CommitmentProof, paths: Set<CommitmentPath>) => boolean
```

아래 함수들은 정의될때 반드시 `verifyMembership`와 `verifyNonMembership`의 결합 조합과 같은 결과를 반환해야 합니다.(효율은 다를 수 있습니다.)

```typescript
batchVerifyMembership(root, proof, items) ===
  all(items.map((item) => verifyMembership(root, proof, item.path, item.value)))
```

```typescript
batchVerifyNonMembership(root, proof, items) ===
  all(items.map((item) => verifyNonMembership(root, proof, item.path)))
```

묶음(batch) 증명이 가능하고, 원소를 각각 증명하는 것보다 더 효율적이라면, commitment 구성은 묶음(batch) 증명 함수를 정의해야 합니다.

### 속성 & 불변 (Properties & Invariants)

Commitments는 반드시 *comlete*, *sound*, 그리고 *position binding*를 갖춰야 합니다. 이런 속성들은 관리자(manager), 증명자(prover), 검증자(verifier)에 의해(그리고 종종 정해진 알고리즘) 승인된 보안 매개변수 `k`와 관련하여 정의됩니다.

#### 완결성 (Completeness)

Commitment 증명은 반드시 *완성*되어야 합니다: commitment에 포함된 경로 => 값 대응관계는 그것이 포함됐다는 것이 항상 증명될 수 있어야 하며, commitment에 포함되지 않은 경로는 그것이 제외됐다는 것이, 보안 매개변수 `k`에 의해 무시되는 경우를 제외하고 항상 증명될 수 있어야 합니다.

commitment `acc`에 포함된 모든 접두사 `prefix`와 모든 경로 `path`, 마지막으로 설정된 값 `value`에 대하여.

```typescript
root = getRoot(acc)
proof = createMembershipProof(acc, applyPrefix(prefix, path), value)
```

```
Probability(verifyMembership(root, proof, applyPrefix(prefix, path), value) === false) negligible in k
```

commitment `acc`에 포함되지 않은 모든 접두사 `prefix`와 모든 경로 `path`, 모든 `proof`의 값과 모든 `value`의 값에 대하여,

```typescript
root = getRoot(acc)
proof = createNonMembershipProof(acc, applyPrefix(prefix, path))
```

```
Probability(verifyNonMembership(root, proof, applyPrefix(prefix, path)) === false) negligible in k
```

#### Soundness

Commitment 증명은 반드시 *합리적(sound)*여야 합니다: 보안 매개변수 `k`에 의해 무시되는 경우를 제외하고, Commitment에 포함되지 않은 경로 =>값 대응관계는 반드시 포함되었다고 증명되거나, commitment에 포함된 경로가 제외더았다고 증명되어서는 안됩니다.

Commitment `acc`의 모든 접두사 `prefix`와 모든 경로 `path`, 마지막으로 설정된 값 `value`, 모든 `proof`의 값에 대하여,

```
Probability(verifyNonMembership(root, proof, applyPrefix(prefix, path)) === true) negligible in k
```

Commitment `acc`의 모든 접두어 `prefix`와 모든 경로 `path`, 모든 `proof`의 값과 모든 `value`의 값에 대하여,

```
Probability(verifyMembership(root, proof, applyPrefix(prefix, path), value) === true) negligible in k
```

#### Position binding

Commitment 증명은 반드시 *position binding*여야 합니다: 어떤 commitment 경로는 반드시 하나의 값(value)로만 대응될 수 있고, 하나의 commitment 증명은 보안 매개변수 k에 의해 무시되는 경우를 제외하고, 같은 경로가 다른 값을 열었다는 것을 증명할 수 없습니다.

Commitment `acc` 의 임의의 접두어 `prefix`와 임의의 경로 set `path`, 단 하나의 `value`에 대하여:

```typescript
root = getRoot(acc)
proof = createMembershipProof(acc, applyPrefix(prefix, path), value)
```

```
Probability(verifyMembership(root, proof, applyPrefix(prefix, path), value) === false) negligible in k
```

`value !== otherValue`를 만족하는 모든 `otherValue`와, 모든 `proof`의 값에 대하여,

```
Probability(verifyMembership(root, proof, applyPrefix(prefix, path), otherValue) === true) negligible in k
```

## 하위 호환성 (Backwards Compatibility)

적용되지 않습니다.

## 상위 호환성 (Forwards Compatibility)

Commitment 알고리즘은 수정될 예정입니다. Versioning 연결 및 채널을 통해 소개될 새로운 알고리즘을 확인할 수 있습니다.

## 예제 구현 (Example Implementation)

곧 구현될 예정입니다.

## 다른 구현 (Other Implementations)

곧 구현될 예정입니다.

## 히스토리

보안 정의는 대부분 백서에서 제공되고, 다소 단순화 되었습니다.

- [Vector Commitments and their Applications](https://eprint.iacr.org/2011/495.pdf)
- [Commitments with Applications to Anonymity-Preserving Revocation](https://eprint.iacr.org/2017/043.pdf)
- [Batching Techniques for Commitments with Applications to IOPs and Stateless Blockchains](https://eprint.iacr.org/2018/1188.pdf)

이 사양에 대한 많은 의견을 주신 Dev Ojha에게 감사드립니다.

2019년 4월 25일 - 초안 작성

## 저작권

이 게시물의 모든 내용은 [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0) 라이센스가 부여됩니다.
