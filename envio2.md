## Uniswap V2 Indexer Setup – Part 2

This part focuses on configuring `config.yaml`, defining your `schema.graphql`, and writing the event handler logic to successfully index Uniswap V2's `PairCreated` and `Swap` events using Envio.

---

### Folder Structure

Your project folder should look like this after initialization:

```
abis/
config.yaml
generated/
node_modules/
schema.graphql
src/
test/
.env
.env.example
.gitignore
.npmrc
README.md
package.json
pnpm-lock.yaml
pnpm-workspace.yaml
tsconfig.json
```

### ABI Requirement

Download the Uniswap V2 Pair ABI from the following URL and place it in the `abis/` folder:

```
https://unpkg.com/@uniswap/v2-core@1.0.0/build/IUniswapV2Pair.json
```

Save the file as `UniswapV2Pair.json` inside the `abis` folder.

---

### Selecting a Start Block

From the [Uniswap V2 deployment docs](https://docs.uniswap.org/contracts/v2/reference/smart-contracts/v2-deployments), use the factory address:

```
0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f
```

Go to [Etherscan](https://etherscan.io/address/0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f), look at the last 50 transactions and find a recent block number. Pick a slightly earlier block to ensure fast indexing during testing. For example:

```
start_block: 22398017
```

---

### `config.yaml`

```
nano config.yaml
```

This file declares what contracts and events Envio should index:

```yaml
name: uniswap-v2-indexer
unordered_multichain_mode: true

networks:
  - id: 1
    start_block: 22398017
    contracts:
      - name: UniswapV2Factory
        address: 0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f
        handler: src/EventHandlers.ts
        events:
          - event: PairCreated(address indexed token0, address indexed token1, address pair, uint256)

      - name: UniswapV2Pair
        abi_file_path: abis/UniswapV2Pair.json
        handler: src/EventHandlers.ts
        events:
          - event: Swap(address indexed sender, uint256 amount0In, uint256 amount1In, uint256 amount0Out, uint256 amount1Out, address indexed to)
```

---

### `schema.graphql`
```
nano schema.graphql
```

This defines how the data will be stored in Hasura:

```graphql
type UniswapV2Factory_PairCreated @sourceEvent {
  id: ID!
  token0: String!
  token1: String!
  pair: String!
  _3: BigInt!
}

type Swap @sourceEvent {
  id: ID!
  sender: String!
  to: String!
  amount0In: BigInt!
  amount1In: BigInt!
  amount0Out: BigInt!
  amount1Out: BigInt!
  blockNumber: BigInt!
  txHash: String!
  timestamp: BigInt!
}
```

---

### Event Handlers: `src/EventHandlers.ts`
```
nano src/EventHandlers.ts
```

This file includes logic for capturing and storing `PairCreated` and `Swap` events:

```ts
import { UniswapV2Factory, UniswapV2Pair } from "../generated/src/Handlers.gen";

UniswapV2Factory.PairCreated.contractRegister(({ event, context }) => {
  context.addUniswapV2Pair(event.params.pair);
  console.log(`Registered new UniswapV2Pair: ${event.params.pair}`);
});

UniswapV2Pair.Swap.handler(async ({ event, context }) => {
  await context.Swap.set({
    id: `${event.chainId}_${event.block.number}_${event.logIndex}`,
    sender: event.params.sender,
    to: event.params.to,
    amount0In: event.params.amount0In,
    amount1In: event.params.amount1In,
    amount0Out: event.params.amount0Out,
    amount1Out: event.params.amount1Out,
    blockNumber: BigInt(event.block.number),
    txHash: event.block.hash,
    timestamp: BigInt(event.block.timestamp),
  });
});
```

---

### Next Steps

After making these changes:

```bash
pnpm codegen
pnpm envio dev
```
![image](https://github.com/user-attachments/assets/530be39b-ab38-41f4-8b9d-a41114d839d7)



Now head to Hasura and check if the `Swap` table is being populated correctly.

➡️ In the next part, we’ll explore how to query and analyze the indexed data.
