# Uniswap V2 Volume Indexer Setup (Using Envio)

### What is Envio?

Envio is a powerful and flexible event indexing framework built in Rust. It captures smart contract events on EVM-compatible chains and syncs them into a Hasura-backed database.

In this guide series, we’ll walk through indexing Swap events from Uniswap V2 and recording their volume data step by step.

---

## 1. Ubuntu Setup & Required Dependencies

For a fresh server or Ubuntu installation, install the following dependencies:

### Node.js, npm, and pnpm

```bash
sudo apt update
sudo apt install curl -y
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt install -y nodejs
npm install -g pnpm
```

### Docker

```bash
sudo apt install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=\"$(dpkg --print-architecture)\" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  \"$(. /etc/os-release && echo \"$VERSION_CODENAME\")\" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Docker Compose (if `docker compose` command doesn’t work)

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/v2.23.3/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### Verify Installations

```bash
docker --version
docker compose version
node -v
pnpm -v
```

---

## 2. Initializing an Envio Project

Create a new folder and initialize the Envio project:

```bash
pnpx envio init
```

### Questions during `pnpx envio init`

You’ll be prompted with several questions. Here are the recommended answers:

```
? Specify a folder name (ENTER to skip):
```

→ Type a folder name or press ENTER to use the current folder. For example: `uniswap-v2-indexer`

```
? Which language would you like to use?
  JavaScript
> TypeScript
  ReScript
```

→ Choose `TypeScript`

```
? Choose blockchain ecosystem
> Evm
  Fuel
```

→ Choose `Evm` since Uniswap runs on Ethereum.

```
? Choose an initialization option
> Contract Import
  Template
```

→ Choose `Contract Import` to import an existing contract.

```
? Would you like to import from a block explorer or a local abi?
> Block Explorer
  Local ABI
```

→ Choose `Block Explorer` if the contract is verified on Etherscan. Otherwise, choose `Local ABI`.

```
? Which blockchain would you like to import a contract from?
> ethereum-mainnet 🥇
```

→ Select `ethereum-mainnet`

```
? What is the address of the contract?
```

→ Paste the Uniswap V2 Factory address from the official docs (https://docs.uniswap.org/contracts/v2/reference/smart-contracts/v2-deployments):
`0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f`

```
? Which events would you like to index?
> [x] PairCreated(address indexed token0, address indexed token1, address pair, uint256)
```

→ Select the `PairCreated` event.

```
? Would you like to add another contract?
> I'm finished
```

→ Choose `I'm finished` to proceed.

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
mkdir abis
curl -o abis/IUniswapV2Pair.json https://unpkg.com/@uniswap/v2-core@1.0.0/build/IUniswapV2Pair.json
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
console.log(`Swap volume at ${event.srcAddress}: ${volume}`);
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

If you see "Registered new UniswapV2Pair" and "Swap volume at" in the logs, it means everything is working correctly.

Now head to Hasura and check if the `Swap` table is being populated correctly.

open in browser ``` yorserver_ipaddress:8080 ```
default password ``` testing ```

![image](https://github.com/user-attachments/assets/02592a0c-ac6a-419c-8ec5-2290fd0a5472)

![image](https://github.com/user-attachments/assets/5a021260-7f1a-455b-9d3f-fe0af8c706b3)

If the Swap table contains data, you’ve made it!

