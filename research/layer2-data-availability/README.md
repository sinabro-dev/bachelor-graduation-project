## How Tx Stored (L2 -> L1)
[Optimism 공식 문서](https://community.optimism.io/docs/protocol/protocol-2.0/#canonicaltransactionchain-ctc)에 들어가보면, 어떤 방식으로 L2의 트랜잭션들을 L1로 저장하는지 대략적으로 기술되어 있다.
타 Layer 2와 비교하여 Rollup이 가지는 강점이 Data Availability이기 때문에 이를 어떻게 구현하였는지 확인해보아야 한다.

위에서 알아본 Calldata라는 공간에 Batch로 L2 트랜잭션들을 기록한다.
예를 들면, [L2 Batch (Optimism Etherscan)](https://optimistic.etherscan.io/batch/158416), [L1 Tx (Etherscan)](https://etherscan.io/tx/0x7a1a7deb40a205e0e56f1434889fe7c1f293a1cea8f677f119e13e0b3a4ab836) 와 같이 L2에서는 Batch 트랜잭션을 보내고, L1에서는 그 데이터를 Calldata에서 받아온다.
아래 캡처 이미지에서 `Input Data`가 Calldata에 해당한다.
![](images/example-ctc-tx.png)

### Encoding/Decoding Spec
Hex 문자열로 인코딩되어 있는데, 이를 디코딩해서 정말 L2 트랜잭션들이 담겨있는지 확인하고자 한다.

이 부분은 시간이 흐름에 따라 Optimism 개발진이 스펙을 변경할 수 있기 때문에 현재 기술한 것과 달라질 수 있다.
> 실제로 Optimism 초기 런칭 때와 글 작성 시점의 인코딩 스펙이 달라서 애를 좀 먹었다..

그나마 권장되는 방법은 공식 GitHub에서 검색해보는 것이다.
지금은 [여기](https://github.com/ethereum-optimism/optimism/blob/%40eth-optimism/sdk%401.6.5/packages/core-utils/src/optimism/batch-encoding.ts)에서 전반적인 과정을 토대로 유추해볼 수 있다.
코드를 읽다보면 `Legacy`라고 표시된 부분이 있으므로 이를 주의해서 살펴보아야 한다.

위의 예제의 Calldata를 디코딩해보면 아래처럼 된다.
| hex        | size [B]         | field                 | description              |
| ---------- | ---------------- | --------------------- | ------------------------ |
| d0f89344   | 4                | sighash               | function signature       |
| 000189226e | 5                | shouldStartAtElement  | starting tx index        |
| 00004b     | 3                | totalElementsToAppend | elements to append       |
| 000006     | 3                | numContexts           | number of batch contexts |
| .          | 16 x numContexts | contexts              | contexts                 |
| .          | var              | transactions          | transactions             |

각 Batch Context는 아래와 유사한 구조를 갖는다. 다만, 가장 먼저 위치한 Context는 Dummy이다. 아래 Value들은 두 번째 Context를 나타낸 것이다.
| hex        | size [B] | field                          | description                |
| ---------- | -------- | ------------------------------ | -------------------------- |
| 000009     | 3        | numSequencedTransactions       | tx sent directly to L2     |
| 000000     | 3        | numSubsequentQueueTransactions | deposits with this context |
| 006332ffbe | 5        | ctxTimestamp                   | timestamp                  |
| 0000ee6b5a | 5        | ctxBlockNumber                 | l1 block number            |
이 경우 6개의 Context가 있으므로, Dummy를 포함하여 위와 같은 Context 정보가 6개 연이어 이어진다.

이후 L2 트랜잭션 정보가 뒤따라오는데, 압축된 후 기록되어서 압축을 푸는 과정이 필요하다.
위에서 링크를 걸어놓은 공식 GitHub 코드 로직을 살펴보면, `zlib.deflateSync()`로 압축하는 것을 확인할 수 있다.
따라서 반대로 `zlib.inflateSync()`로 압축을 풀면 된다.

1차적으로 아래의 구조로 데이터가 담겨있다.
| hex         | size [B]     | field        | description    |
| ----------- | ------------ | ------------ | -------------- |
| 0000ec      | 3            | txDataLength | tx data length |
| f8ea...9800 | txDataLength | txData       | tx data        |

그리고 각 txData에는 다음의 필드들을 갖는다. (일반적으로 생각하는 이더리움 트랜잭션이 갖는 그것과 유사하다.)
- chainId
- hash
- nonce
- from
- to
- value
- gasLimit
- gasPrice
- type
- data
- r
- s
- v
일부 필드 값의 크기는 가변적이기에 txDataLength 필드가 필요하다.

참고로, 직접 디코딩하는 코드는 `scratch/` 경로에 올려놓았다.

