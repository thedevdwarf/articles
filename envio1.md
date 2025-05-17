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

