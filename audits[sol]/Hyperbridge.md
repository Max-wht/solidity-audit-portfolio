# M-01 - A reverting inbound message permanently bricks the LayerZero OFT channel

cofinder: https://x.com/Viraz04

## Description

The `onAccept` function requires every inbound nonce to equal the channel's current nonce plus one. It writes the new nonce and then calls the destination OApp with a normal external call. If the `lzReceive` function reverts thenthe nonce too does not get incremented.

The ISMP host consequently deletes the failed receipt and permits the same message to be retried indefinitely. Meanwhile, `skip`, `clear`, `nilify`, and `burn` functions are not implemented, so neither the OApp nor an administrator can advance or repair the channel.

A deterministically reverting message at nonce `N` therefore leaves the channel at `N - 1`. The poisoned message continues to fail, while every later message fails with `InvalidNonce`.

## Impact

A single malformed or permanently unexecutable OFT message can permanently deny service to its `(receiver, srcEid, sender)` channel. Tokens locked or burned by the source-chain cannot be credited on the destination as the channel remains stuck.

## Proof of Concept

The `test_StuckInboundNoncePermanentlyBricksOFTAdapterChannel` fork test in `sdk/packages/lz-endpoint/test/HyperbridgeLzEndpointTest.sol` uses the deployed Ethereum ISMP host and mainnet USDC. It submits a zero-recipient OFT payload at nonce 1, confirms that the live host deletes its failed receipt and proves that an honest nonce-2 transfer remains locked on the source and uncredited on the destination.

```bash
MAINNET_FORK_URL=https://ethereum-rpc.publicnode.com forge test \
  --match-test test_StuckInboundNoncePermanentlyBricksOFTAdapterChannel -vvv
```

## Recommendation

The following function-specific diff shows exactly where each change is applied.

### 1. Add failed-payload and recovery state

```diff
 contract HyperbridgeLzEndpoint is HyperApp, Ownable, Pausable, ILayerZeroEndpointV2 {
     error InvalidNonce(uint64 expected, uint64 got);
+    error UnauthorizedRecovery();
+    error InvalidPayloadHash();
+    error PayloadNotNilified();
+
+    bytes32 internal constant NIL_PAYLOAD_HASH = bytes32(type(uint256).max);

     mapping(address => mapping(uint32 => mapping(bytes32 => uint64))) internal _inboundNonce;

+    mapping(address => mapping(uint32 => mapping(bytes32 => mapping(uint64 => bytes32))))
+        internal _inboundPayloadHashes;
+
+    // OApp => account authorized to perform recovery
+    mapping(address => address) internal _delegates;
+
+    event OAppDeliveryFailed(
+        address indexed receiver,
+        uint32 indexed srcEid,
+        bytes32 indexed sender,
+        uint64 nonce,
+        bytes32 payloadHash
+    );
 }
```

### 2. Update the `onAccept` function

```diff
 function onAccept(IncomingPostRequest calldata incoming)
     external
     override
     onlyHost
     whenNotPaused
 {
     // Existing source, payload, and nonce validation remains unchanged.
     if (nonce != expectedNonce) revert InvalidNonce(expectedNonce, nonce);

+    // The packet is accepted after host verification, independently of OApp execution.
     _inboundNonce[receiverAddr][srcEid][sender] = nonce;

     Origin memory origin = Origin({srcEid: srcEid, sender: sender, nonce: nonce});
-    ILayerZeroReceiver(receiverAddr).lzReceive(origin, guid, message, address(0), "");
+    try ILayerZeroReceiver(receiverAddr).lzReceive(origin, guid, message, address(0), "") {
+        // Successful execution requires no retained payload state.
+    } catch {
+        bytes32 payloadHash = keccak256(abi.encode(guid, message));
+        _inboundPayloadHashes[receiverAddr][srcEid][sender][nonce] = payloadHash;
+        emit OAppDeliveryFailed(receiverAddr, srcEid, sender, nonce, payloadHash);
+    }
 }
```

### 3. Add `retryPayload` generic function for retrying failed payloads

```diff
+function retryPayload(
+    address receiver,
+    Origin calldata origin,
+    bytes32 guid,
+    bytes calldata message
+) external {
+    bytes32 stored =
+        _inboundPayloadHashes[receiver][origin.srcEid][origin.sender][origin.nonce];
+    bytes32 supplied = keccak256(abi.encode(guid, message));
+
+    if (stored == bytes32(0) || stored == NIL_PAYLOAD_HASH || stored != supplied) {
+        revert InvalidPayloadHash();
+    }
+
+    // If lzReceive reverts again, this deletion is rolled back automatically.
+    delete _inboundPayloadHashes[receiver][origin.srcEid][origin.sender][origin.nonce];
+    ILayerZeroReceiver(receiver).lzReceive(origin, guid, message, msg.sender, "");
+}
```

### 4. Update `setDelegate` function and add recovery authorization

```diff
-function setDelegate(address) external pure override {}
+function setDelegate(address delegate) external override {
+    _delegates[msg.sender] = delegate;
+}
+
+modifier onlyOAppOrDelegate(address oapp) {
+    if (msg.sender != oapp && msg.sender != _delegates[oapp]) {
+        revert UnauthorizedRecovery();
+    }
+    _;
+}
```

### 5. Update `clear`, `skip`, `nilify` and `burn` functions

```diff
-function clear(address, Origin calldata, bytes32, bytes calldata) external pure override {}
+function clear(
+    address oapp,
+    Origin calldata origin,
+    bytes32 guid,
+    bytes calldata message
+) external override onlyOAppOrDelegate(oapp) {
+    bytes32 supplied = keccak256(abi.encode(guid, message));
+    bytes32 stored =
+        _inboundPayloadHashes[oapp][origin.srcEid][origin.sender][origin.nonce];
+    if (stored != supplied) revert InvalidPayloadHash();
+
+    delete _inboundPayloadHashes[oapp][origin.srcEid][origin.sender][origin.nonce];
+}

-function skip(address, uint32, bytes32, uint64) external pure override {}
+function skip(
+    address oapp,
+    uint32 srcEid,
+    bytes32 sender,
+    uint64 nonce
+) external override onlyOAppOrDelegate(oapp) {
+    uint64 expected = _inboundNonce[oapp][srcEid][sender] + 1;
+    if (nonce != expected) revert InvalidNonce(expected, nonce);
+
+    _inboundNonce[oapp][srcEid][sender] = nonce;
+}


-function nilify(address, uint32, bytes32, uint64, bytes32) external pure override {}
+function nilify(
+    address oapp,
+    uint32 srcEid,
+    bytes32 sender,
+    uint64 nonce,
+    bytes32 payloadHash
+) external override onlyOAppOrDelegate(oapp) {
+    if (_inboundPayloadHashes[oapp][srcEid][sender][nonce] != payloadHash) {
+        revert InvalidPayloadHash();
+    }
+
+    _inboundPayloadHashes[oapp][srcEid][sender][nonce] = NIL_PAYLOAD_HASH;
+}

-function burn(address, uint32, bytes32, uint64, bytes32) external pure override {}
+function burn(
+    address oapp,
+    uint32 srcEid,
+    bytes32 sender,
+    uint64 nonce,
+    bytes32 payloadHash
+) external override onlyOAppOrDelegate(oapp) {
+    if (payloadHash != NIL_PAYLOAD_HASH) revert InvalidPayloadHash();
+    if (_inboundPayloadHashes[oapp][srcEid][sender][nonce] != NIL_PAYLOAD_HASH) {
+        revert PayloadNotNilified();
+    }
+
+    delete _inboundPayloadHashes[oapp][srcEid][sender][nonce];
+}

```

### 6. Update `inboundPayloadHash` function for recovery purposes

```diff
-function inboundPayloadHash(address, uint32, bytes32, uint64)
-    external
-    pure
-    override
-    returns (bytes32)
-{
-    return bytes32(0);
-}
+function inboundPayloadHash(
+    address receiver,
+    uint32 srcEid,
+    bytes32 sender,
+    uint64 nonce
+) external view override returns (bytes32) {
+    return _inboundPayloadHashes[receiver][srcEid][sender][nonce];
+}
```