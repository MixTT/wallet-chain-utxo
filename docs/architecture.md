## Architecture Overview

This repository implements a gRPC gateway that exposes a unified UTXO-chain RPC interface for multiple networks: Bitcoin, BitcoinCash, Dash, Dogecoin, Litecoin, and Zen. The service is written in Go, is stateless, and can be horizontally scaled. Chain-specific logic is encapsulated behind adaptors that implement a shared interface.

### Tech Stack
- Go 1.22+
- gRPC + Protocol Buffers (service: `WalletUtxoService` in `rpc/utxo`)
- Logging via `github.com/ethereum/go-ethereum/log`
- UTXO utilities and node RPC via `btcsuite` packages
- External data via `dapplink-labs/chain-explorer-api` (Oklink adaptor) and chain-specific third-party APIs
- Build and deploy: `Makefile`, multi-stage `Dockerfile`, Helm chart under `charts/`

### High-Level Components
- gRPC API definitions: `rpc/utxo`, `rpc/common`
- Server entrypoint: `main.go` (starts gRPC server and registers service)
- Request interceptor: unary interceptor for logging and panic recovery
- Chain dispatcher: `chaindispatcher/dispatcher.go` routes requests to the correct chain adaptor based on `request.chain`
- Chain adaptor interface: `chain/IChainAdaptor` defines the chain operations contract
- Chain adaptors: implementations under `chain/<chain>` (e.g., `bitcoin`, `litecoin`, `zen`, etc.)
- Base clients:
  - `chain/base/BaseClient`: JSON-RPC client to UTXO nodes (btcd-compatible)
  - `chain/base/BaseDataClient`: typed client for Oklink-based explorer API
- Configuration:
  - Types: `config/config.go`
  - Default file: `config.yml` (server, network, chains, per-chain endpoints/keys)
- Deployment artifacts: `Dockerfile`, Helm chart in `charts/` and `values.yaml`

### System Diagram
```mermaid
flowchart LR
  Client -->|gRPC| GRPC[WalletUtxoService]
  GRPC -->|UnaryInterceptor| Interceptor
  Interceptor --> Dispatcher
  Dispatcher -->|select by request.chain| BTC[Bitcoin Adaptor]
  Dispatcher --> LTC[Litecoin Adaptor]
  Dispatcher --> ZEN[Zen Adaptor]
  Dispatcher --> BCH[BitcoinCash Adaptor]
  Dispatcher --> DASH[Dash Adaptor]
  Dispatcher --> DOGE[Dogecoin Adaptor]

  BTC -->|JSON-RPC (btcd)| BTCNode[Bitcoin Core RPC]
  BTC -->|Data API| Oklink
  LTC -->|JSON-RPC (btcd)| LTCNode[Litecoin RPC]
  LTC --> Oklink
  ZEN -->|Explorer API| HorizenExplorer
```

## Runtime Flow
1. Process starts in `main.go`.
   - Reads configuration from `-c <path>` (default `config.yml`).
   - Constructs `ChainDispatcher` with a registry of chain adaptors based on `config.Chains`.
   - Starts a gRPC server (with a unary interceptor) on `:<server.port>` and registers `WalletUtxoService` with the dispatcher as handler.
2. Each gRPC call passes through the interceptor for logging and panic safety.
3. The dispatcher determines the target chain from the request payload, validates support, then delegates to the corresponding adaptor.
4. The adaptor performs chain-specific work using:
   - Node RPC via `BaseClient` (btcd-compatible JSON-RPC)
   - Explorer/Data API via `BaseDataClient` (Oklink) or a chain-specific third-party client
5. Responses are mapped to the protobuf response types with `common.ReturnCode` and returned to the client.

## Configuration
- File: `config.yml`
- Keys:
  - `server`: `port`, `host`
  - `network`: `mainnet` | `testnet` | `regtest` (string)
  - `chains`: list of enabled chain names (e.g., `Bitcoin`, `Dash`, `BitcoinCash`, `Litecoin`, `Zen`)
  - `walletnode`: per-chain endpoints and credentials
    - `rpc_url`, `rpc_user`, `rpc_pass`
    - `data_api_url`, `data_api_key`, `data_api_token`
    - `tp_api_url` (third-party API), `time_out`

Notes:
- Only chains present in `chains` will be registered in the dispatcher at startup.
- Treat `rpc_pass`, `data_api_key` and similar values as secrets. In production, inject via environment/secrets and mount into the container rather than committing to VCS.

## gRPC API Surface (WalletUtxoService)
Defined in `rpc/utxo`.
- Support and address utilities:
  - `getSupportChains`, `convertAddress`, `validAddress`
- Fees and account state:
  - `getFee`, `getAccount`, `getUnspentOutputs`
- Blocks and headers:
  - `getBlockByNumber`, `getBlockByHash`, `getBlockHeaderByHash`, `getBlockHeaderByNumber`
- Transactions:
  - `SendTx`, `getTxByAddress`, `getTxByHash`
  - `createUnSignTransaction`, `buildSignedTransaction`, `decodeTransaction`, `verifySignedTransaction`

## Chain Adaptors
All adaptors implement `chain.IChainAdaptor` to provide a consistent API. Highlights:

### Bitcoin (`chain/bitcoin`)
- Node access: `BaseClient` via `rpcclient` (btcd-compatible)
- Data access: `BaseDataClient` (Oklink) and a third-party blockchain API client
- Implements address conversion/validation, fees, account state, UTXO queries, block/tx queries
- Transaction flows:
  - Create unsigned tx: compute sign hashes and serialized tx skeleton
  - Build signed tx: verify/signature wiring and script building; serialize and return raw tx
  - Broadcast via node RPC `sendrawtransaction`

### Litecoin (`chain/litecoin`)
- Node access: `BaseClient`
- Data access: `BaseDataClient` (Oklink)
- Implements address conversion/validation, fees, account state, UTXO queries, send/broadcast, address/tx listing
- Limitations: several block/header and signing helpers are marked TODO and not yet implemented.

### Zen (`chain/zen`)
- Data access: Horizen explorer API via custom client
- Implements balance, fees, UTXO queries, block and tx queries, and raw tx signing/building tailored to Horizen
- Some header/decoder methods are marked not supported by design

### Other Chains
- `bitcoincash`, `dash`, `dogecoin` packages follow the same adaptor pattern. Enable them by adding to `config.chains` and ensuring their endpoints are configured under `walletnode`.

## Dispatcher and Interceptor
- Dispatcher: Builds a registry of chain name (lowercased) to adaptor factory at startup. Rejects unsupported chains with `UnsupportedOperation` messages.
- Interceptor: Logs method name, chain, request, and provides panic recovery returning a gRPC `Internal` status code.

## Error Handling & Logging
- Logging via `go-ethereum/log`, default terminal handler bound at configuration load.
- API responses encode success/failure with `common.ReturnCode` and a human-readable `Msg`.
- Panics are caught by the interceptor and surfaced as gRPC errors.

## Build, Run, and Deploy
### Local
```bash
go mod tidy
go build
./wallet-chain-utxo -c ./config.yml
# Optional: grpcui
grpcui -plaintext 127.0.0.1:8389
```

### Container Image
- Multi-stage build in `Dockerfile` builds the binary with `make` and copies `wallet-chain-utxo` into a minimal Alpine image.
- Entrypoint: `wallet-chain-utxo`; default command `-c /etc/wallet-chain-utxo/config.yml`.

### Kubernetes (Helm)
- Chart under `charts/` with service, deployment, HPA, and ingress templates.
- `values.yaml` exposes the gRPC port (container `targetPort` 8389) via NodePort `30009` by default. Adjust `image.repository`, `image.tag`, and ports for your setup.

## Security Considerations
- Node RPC client disables TLS by default (`DisableTLS: true`). Use TLS/secure transport for production nodes if available.
- gRPC reflection is enabled; disable or restrict in production environments.
- Do not commit secrets. Pass RPC credentials and API keys via environment variables or K8s Secrets mounted at runtime.

## Observability
- Built-in structured logging. No metrics/tracing by default; integrate Prometheus/OpenTelemetry as needed.

## Extensibility: Adding a New Chain
1. Create `chain/<newchain>` package with a `ChainName` constant and an adaptor that implements `chain.IChainAdaptor`.
2. Register the adaptor factory in `chaindispatcher.New` and ensure its chain name is added to `supportedChains`.
3. Add configuration under `walletnode.<short>` in `config.yml` and include the chain in `chains`.
4. Implement node/data clients analogous to `BaseClient` and/or `BaseDataClient` as appropriate.

## Known Limitations / TODOs
- Litecoin adaptor: several block/header and signing-related methods are unimplemented (panic placeholders).
- Configuration for Litecoin adaptor in code currently references BTC fields; align to `walletnode.ltc` before production use.
- The `network` setting exists in config but some adaptors assume mainnet parameters; wire up testnet/regtest handling when needed.

## Repository Map
- `main.go`: gRPC server bootstrap and service registration
- `chaindispatcher/dispatcher.go`: chain routing and interceptor
- `chain/chainadaptor.go`: adaptor interface (contract)
- `chain/base/*`: shared node and data explorer clients
- `chain/<chain>/*`: per-chain adaptor logic
- `config/config.go`, `config.yml`: configuration types and default settings
- `rpc/utxo`, `rpc/common`: protobuf-generated types and service stubs
- `Dockerfile`, `Makefile`: build and containerization
- `charts/`, `values.yaml`: Helm chart and default values

