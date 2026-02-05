# Profen Python SDK — Documentation Overview

This overview summarizes the system architecture, contract bindings, error codes, and upgrade strategy. It is a starting point for contributors and integrators.

## 1) Architecture

### High-level modules

- SDK public surface: [sdk/](../sdk/)
  - Main entrypoints: [sdk/sdk.py](../sdk/sdk.py) and [sdk/sdk_ipc.py](../sdk/sdk_ipc.py)
  - Core config/connection: [sdk/config.py](../sdk/config.py), [sdk/connection.py](../sdk/connection.py)
  - Models shared across APIs: [sdk/model.py](../sdk/model.py)
  - Streaming API: [sdk/sdk_streaming.py](../sdk/sdk_streaming.py)
- IPC (blockchain) bindings and contracts: [private/ipc/](../private/ipc/)
  - Contract wrappers (auto-generated bindings): [private/ipc/contracts/](../private/ipc/contracts/)
  - Error mapping: [private/ipc/errors.py](../private/ipc/errors.py)
- Encryption and chunking: [private/encryption/](../private/encryption/)
- gRPC protobufs: [private/pb/](../private/pb/)
- Tests: [tests/](../tests/)

### Data flow (conceptual)

1. User code instantiates `SDK` with `SDKConfig` (node address, IPC node, keys).
2. For streaming operations, requests go through the gRPC connection layer.
3. For IPC operations, the SDK routes to IPC clients and contract bindings.
4. Errors from EVM contracts are mapped to user-friendly names.

### Ownership boundaries

- `sdk/*` is public, stable API.
- `private/*` is internal and may change as IPC / contract bindings evolve.

## 2) Contract specifications

Contract bindings are auto-generated Python wrappers under [private/ipc/contracts/](../private/ipc/contracts/). Each file includes the ABI and bytecode for deployments (where applicable). The main contract wrappers are:

- `StorageContract`: [private/ipc/contracts/storage.py](../private/ipc/contracts/storage.py)
- `AccessManagerContract`: [private/ipc/contracts/access_manager.py](../private/ipc/contracts/access_manager.py)
- `PolicyFactoryContract`: [private/ipc/contracts/policy_factory.py](../private/ipc/contracts/policy_factory.py)
- `ListPolicyContract`: [private/ipc/contracts/list_policy.py](../private/ipc/contracts/list_policy.py)
- `PDPVerifier`: [private/ipc/contracts/pdp_verifier.py](../private/ipc/contracts/pdp_verifier.py)
- `SinkContract`: [private/ipc/contracts/sink.py](../private/ipc/contracts/sink.py)
- Upgrade proxy: `ERC1967Proxy`: [private/ipc/contracts/erc1967_proxy.py](../private/ipc/contracts/erc1967_proxy.py)

### ABI and bytecode sources

- Each contract wrapper contains `MetaData.ABI` and, when deployable, `MetaData.BIN`.
- When updating contracts, regenerate wrappers and verify that ABI signatures and error selectors align with on-chain deployments.

## 3) Error codes

IPC error selectors are mapped to human-readable names in [private/ipc/errors.py](../private/ipc/errors.py). The helper `error_hash_to_error()` extracts the 4‑byte selector from an EVM revert and maps it to a descriptive error.

### Current selector map

| Selector | Error name |
|---|---|
| 0x497ef2c2 | BucketAlreadyExists |
| 0x4f4b202a | BucketInvalid |
| 0xdc64d0ad | BucketInvalidOwner |
| 0x938a92b7 | BucketNonexists |
| 0x89fddc00 | BucketNonempty |
| 0x6891dde0 | FileAlreadyExists |
| 0x77a3cbd8 | FileInvalid |
| 0x21584586 | FileNonexists |
| 0xc4a3b6f1 | FileNonempty |
| 0xd09ec7af | FileNameDuplicate |
| 0xd96b03b1 | FileFullyUploaded |
| 0x702cf740 | FileChunkDuplicate |
| 0xc1edd16a | BlockAlreadyExists |
| 0xcb20e88c | BlockInvalid |
| 0x15123121 | BlockNonexists |
| 0x856b300d | InvalidArrayLength |
| 0x17ec8370 | InvalidFileBlocksCount |
| 0x5660ebd2 | InvalidLastBlockSize |
| 0x1b6fdfeb | InvalidEncodedSize |
| 0xfe33db92 | InvalidFileCID |
| 0x37c7f255 | IndexMismatch |
| 0xcefa6b05 | NoPolicy |
| 0x5c371e92 | FileNotFilled |
| 0xdad01942 | BlockAlreadyFilled |
| 0x4b6b8ec8 | ChunkCIDMismatch |
| 0x0d6b18f0 | NotBucketOwner |
| 0xc4c1a0c5 | BucketNotFound |
| 0x3bcbb0de | FileDoesNotExist |
| 0xa2c09fea | NotThePolicyOwner |
| 0x94289054 | CloneArgumentsTooLong |
| 0x4ca249dc | Create2EmptyBytecode |
| 0xf3714a9b | ECDSAInvalidSignatureS |
| 0x367e2e27 | ECDSAInvalidSignatureLength |
| 0xf645eedf | ECDSAInvalidSignature |
| 0xb73e95e1 | AlreadyWhitelisted |
| 0xe6c4247b | InvalidAddress |
| 0x584a7938 | NotWhitelisted |
| 0x227bc153 | MathOverflowedMulDiv |
| 0xe7b199a6 | InvalidBlocksAmount |
| 0x59b452ef | InvalidBlockIndex |
| 0x55cbc831 | LastChunkDuplicate |
| 0x2abde339 | FileNotExists |
| 0x48e0ed68 | NotSignedByBucketOwner |
| 0x923b8cbb | NonceAlreadyUsed |
| 0x9605a010 | OffsetOutOfBounds |

### Guidance for adding new errors

- Add new selectors to `error_map` and add tests that ensure the mapping is stable.
- If errors originate from new contracts, verify selector values from the ABI.

## 4) Upgrade strategy

### Contract upgrade path

- The project uses the ERC‑1967 proxy pattern. See [private/ipc/contracts/erc1967_proxy.py](../private/ipc/contracts/erc1967_proxy.py) for the ABI and `Upgraded` event.
- Upgrades should replace implementation addresses while keeping proxy addresses stable.

### Recommended process

1. Deploy new implementation contracts.
2. Verify ABI compatibility for user‑facing methods.
3. Execute proxy upgrade transactions (admin/owner on-chain).
4. Observe `Upgraded` events and confirm new implementation addresses.
5. Regenerate contract bindings if ABI changed.
6. Update error selectors and tests.

### Backwards compatibility

- Avoid breaking changes in public SDK signatures under [sdk/](../sdk/).
- If an API change is unavoidable, bump SDK version and document migration in README.

## 5) Suggested additions

- Add a “Contracts Version Matrix” section linking contract versions to SDK versions.
- Add a “Network/Environment matrix” (mainnet/testnet/local) and defaults.
- Add “Security considerations” for key management and upgrade authority.
- Add a “Data integrity flow” diagram for chunking, erasure coding, and CID validation.
