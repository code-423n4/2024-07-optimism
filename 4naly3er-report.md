# Report


## Gas Optimizations


| |Issue|Instances|
|-|:-|:-:|
| [GAS-1](#GAS-1) | `a = a + b` is more gas effective than `a += b` for state variables (excluding arrays and mappings) | 6 |
| [GAS-2](#GAS-2) | Use assembly to check for `address(0)` | 8 |
| [GAS-3](#GAS-3) | Using bools for storage incurs overhead | 5 |
| [GAS-4](#GAS-4) | Use calldata instead of memory for function arguments that do not get mutated | 2 |
| [GAS-5](#GAS-5) | For Operations that will not overflow, you could use unchecked | 204 |
| [GAS-6](#GAS-6) | Use Custom Errors instead of Revert Strings to save Gas | 10 |
| [GAS-7](#GAS-7) | State variables only set in the constructor should be declared `immutable` | 13 |
| [GAS-8](#GAS-8) | Functions guaranteed to revert when called by normal users can be marked `payable` | 2 |
| [GAS-9](#GAS-9) | `++i` costs less gas compared to `i++` or `i += 1` (same for `--i` vs `i--` or `i -= 1`) | 5 |
| [GAS-10](#GAS-10) | Using `private` rather than `public` for constants, saves gas | 8 |
| [GAS-11](#GAS-11) | Use shift right/left instead of division/multiplication if possible | 19 |
| [GAS-12](#GAS-12) | `uint256` to `bool` `mapping`: Utilizing Bitmaps to dramatically save on Gas | 2 |
| [GAS-13](#GAS-13) | Increments/decrements can be unchecked in for-loops | 3 |
| [GAS-14](#GAS-14) | Use != 0 instead of > 0 for unsigned integer comparison | 2 |
| [GAS-15](#GAS-15) | `internal` functions not called by the contract should be removed | 13 |
| [GAS-16](#GAS-16) | WETH address definition can be use directly | 1 |
### <a name="GAS-1"></a>[GAS-1] `a = a + b` is more gas effective than `a += b` for state variables (excluding arrays and mappings)
This saves **16 gas per instance.**

*Instances (6)*:
```solidity
File: src/cannon/MIPS.sol

174:                     sz += 4096 - (sz & 4095);

178:                     state.heap += sz;

234:                     state.preimageOffset += uint32(datLen);

697:             state.step += 1;

751:                 rs += SE(insn & 0xFFFF, 16);

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/MIPS.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

854:         credit[_recipient] += bond;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="GAS-2"></a>[GAS-2] Use assembly to check for `address(0)`
*Saves 6 gas per instance*

*Instances (8)*:
```solidity
File: src/dispute/DisputeGameFactory.sol

97:         if (address(impl) == address(0)) revert NoImplementation(_gameType);

130:         _disputeGameList.push(id);

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

176:         if (root.raw() == bytes32(0)) revert AnchorRootNotFound();

307:         if (parent.counteredBy != address(0)) revert DuplicateStep();

549:         status_ = claimData[0].counteredBy == address(0) ? GameStatus.DEFENDER_WINS : GameStatus.CHALLENGER_WINS;

586:             address recipient = counteredBy == address(0) ? subgameRootClaim.claimant : counteredBy;

621:             if (claim.counteredBy == address(0) && checkpoint.leftmostPosition.raw() > claim.position.raw()) {

652:                 _distributeBond(countered == address(0) ? subgameRootClaim.claimant : countered, subgameRootClaim);

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="GAS-3"></a>[GAS-3] Using bools for storage incurs overhead
Use uint256(1) and uint256(2) for true/false to avoid a Gwarmaccess (100 gas), and to avoid Gsset (20000 gas) when changing from ‘false’ to ‘true’, after having been ‘true’ in the past. See [source](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/58f635312aa21f947cae5f8578638a85aa2519f5/contracts/security/ReentrancyGuard.sol#L23-L27).

*Instances (5)*:
```solidity
File: src/cannon/PreimageOracle.sol

44:     mapping(bytes32 => mapping(uint256 => bool)) public preimagePartOk;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

85:     bool internal initialized;

88:     bool public l2BlockNumberChallenged;

101:     mapping(Hash => bool) public claims;

107:     mapping(uint256 => bool) public resolvedSubgames;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="GAS-4"></a>[GAS-4] Use calldata instead of memory for function arguments that do not get mutated
When a function with a `memory` array is called externally, the `abi.decode()` step has to use a for-loop to copy each index of the `calldata` to the `memory` index. Each iteration of this for-loop costs at least 60 gas (i.e. `60 * <mem_array>.length`). Using `calldata` directly bypasses this loop. 

If the array is passed to an `internal` function which passes the array to another internal function where the array is modified and therefore `memory` is used in the `external` call, it's still more gas-efficient to use `calldata` when the `external` function uses modifiers, since the modifiers may prevent the internal functions from being called. Structs have the same overhead as an array of length one. 

 *Saves 60 gas per instance*

*Instances (2)*:
```solidity
File: src/cannon/PreimageOracle.sol

571:         LibKeccak.StateMatrix memory _stateMatrix,

643:         LibKeccak.StateMatrix memory _stateMatrix,

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

### <a name="GAS-5"></a>[GAS-5] For Operations that will not overflow, you could use unchecked

*Instances (204)*:
```solidity
File: src/cannon/MIPS.sol

4: import { ISemver } from "src/universal/ISemver.sol";

5: import { IPreimageOracle } from "./interfaces/IPreimageOracle.sol";

6: import { PreimageKeyLib } from "./PreimageKeyLib.sol";

77:             bool isSigned = (_dat >> (_idx - 1)) != 0;

78:             uint256 signed = ((1 << (32 - _idx)) - 1) << _idx;

79:             uint256 mask = (1 << _idx) - 1;

89:             function copyMem(from, to, size) -> fromOut, toOut {

103:             from, to := copyMem(from, to, 32) // memRoot

104:             from, to := copyMem(from, to, 32) // preimageKey

105:             from, to := copyMem(from, to, 4) // preimageOffset

106:             from, to := copyMem(from, to, 4) // pc

107:             from, to := copyMem(from, to, 4) // nextPC

108:             from, to := copyMem(from, to, 4) // lo

109:             from, to := copyMem(from, to, 4) // hi

110:             from, to := copyMem(from, to, 4) // heap

112:             from, to := copyMem(from, to, 1) // exitCode

114:             from, to := copyMem(from, to, 1) // exited

115:             from, to := copyMem(from, to, 8) // step

116:             from := add(from, 32) // offset to registers

174:                     sz += 4096 - (sz & 4095);

178:                     state.heap += sz;

207:                     uint32 mem = readMem(a1 & 0xFFffFFfc, 1); // mask the addr to align it to 4 bytes

218:                         let alignment := and(a1, 3) // the read might not start at an aligned address

219:                         let space := sub(4, alignment) // remaining space in memory word

220:                         if lt(space, datLen) { datLen := space } // if less space than data, shorten data

221:                         if lt(a2, datLen) { datLen := a2 } // if requested to read less, read less

222:                         dat := shr(sub(256, mul(datLen, 8)), dat) // right-align data

223:                         dat := shl(mul(sub(sub(4, datLen), alignment), 8), dat) // position data to insert into memory

225:                         let mask := sub(shl(mul(sub(4, alignment), 8), 1), 1) // mask all bytes after start

226:                         let suffixMask := sub(shl(mul(sub(sub(4, alignment), datLen), 8), 1), 1) // mask of all bytes

228:                         mask := and(mask, not(suffixMask)) // reduce mask to just cover the data we insert

229:                         mem := or(and(mem, not(mask)), dat) // clear masked part of original memory, and insert data

234:                     state.preimageOffset += uint32(datLen);

252:                     v0 = a2; // tell program we have written everything

256:                     uint32 mem = readMem(a1 & 0xFFffFFfc, 1); // mask the addr to align it to 4 bytes

262:                         let alignment := and(a1, 3) // the read might not start at an aligned address

263:                         let space := sub(4, alignment) // remaining space in memory word

264:                         if lt(space, a2) { a2 := space } // if less space than data, shorten data

265:                         key := shl(mul(a2, 8), key) // shift key, make space for new info

266:                         let mask := sub(shl(mul(a2, 8), 1), 1) // mask for extracting value from memory

267:                         mem := and(shr(mul(sub(space, a2), 8), mem), mask) // align value to right, mask it

268:                         key := or(key, mem) // insert into key

273:                     state.preimageOffset = 0; // reset offset, to read new pre-image data from the start

288:                         v0 = 0; // O_RDONLY

290:                         v0 = 1; // O_WRONLY

297:                     v1 = EINVAL; // cmd not recognized by this kernel

307:             state.nextPC = state.nextPC + 4;

329:             if (state.nextPC != state.pc + 4) {

367:                 state.nextPC = prevPC + 4 + (SE(_insn & 0xFFFF, 16) << 2);

369:                 state.nextPC = state.nextPC + 4;

411:                 uint64 acc = uint64(int64(int32(_rs)) * int64(int32(_rt)));

417:                 uint64 acc = uint64(uint64(_rs) * uint64(_rt));

429:                 state.lo = uint32(int32(_rs) / int32(_rt));

439:                 state.lo = _rs / _rt;

449:             state.nextPC = state.nextPC + 4;

468:             if (state.nextPC != state.pc + 4) {

479:                 state.registers[_linkReg] = prevPC + 8;

510:             state.nextPC = state.nextPC + 4;

524:             offset_ = 420 + (uint256(_proofIndex) * (28 * 32));

529:             require(s >= (offset_ + 28 * 32), "check that there is enough calldata");

552:                 function hashPair(a, b) -> h {

610:                 function hashPair(a, b) -> h {

663:                 function putField(callOffset, memOffset, size) -> callOffsetOut, memOffsetOut {

672:                 let c := _stateData.offset // calldata offset

673:                 let m := 0x80 // mem offset

674:                 c, m := putField(c, m, 32) // memRoot

675:                 c, m := putField(c, m, 32) // preimageKey

676:                 c, m := putField(c, m, 4) // preimageOffset

677:                 c, m := putField(c, m, 4) // pc

678:                 c, m := putField(c, m, 4) // nextPC

679:                 c, m := putField(c, m, 4) // lo

680:                 c, m := putField(c, m, 4) // hi

681:                 c, m := putField(c, m, 4) // heap

682:                 c, m := putField(c, m, 1) // exitCode

683:                 c, m := putField(c, m, 1) // exited

684:                 c, m := putField(c, m, 8) // step

687:                 mstore(m, add(m, 32)) // offset to registers

697:             state.step += 1;

701:             uint32 opcode = insn >> 26; // 6-bits

711:             uint32 rs; // source register 1 value

712:             uint32 rt; // source register 2 / temp value

751:                 rs += SE(insn & 0xFFFF, 16);

763:             uint32 val = execute(insn, rs, rt, mem) & 0xffFFffFF; // swr outputs more than 4 bytes without the mask

765:             uint32 func = insn & 0x3f; // 6-bits

811:             uint32 opcode = insn >> 26; // 6-bits

814:                 uint32 func = insn & 0x3f; // 6-bits

845:                     return SE(rt >> shamt, 32 - shamt);

857:                     return SE(rt >> rs, 32 - rs);

921:                     return (rs + rt);

925:                     return (rs + rt);

929:                     return (rs - rt);

933:                     return (rs - rt);

964:                     uint32 func = insn & 0x3f; // 6-bits

967:                         return uint32(int32(rs) * int32(rt));

976:                             i++;

988:                     return SE((mem >> (24 - (rs & 3) * 8)) & 0xFF, 8);

992:                     return SE((mem >> (16 - (rs & 2) * 8)) & 0xFFFF, 16);

996:                     uint32 val = mem << ((rs & 3) * 8);

997:                     uint32 mask = uint32(0xFFFFFFFF) << ((rs & 3) * 8);

1006:                     return (mem >> (24 - (rs & 3) * 8)) & 0xFF;

1010:                     return (mem >> (16 - (rs & 2) * 8)) & 0xFFFF;

1014:                     uint32 val = mem >> (24 - (rs & 3) * 8);

1015:                     uint32 mask = uint32(0xFFFFFFFF) >> (24 - (rs & 3) * 8);

1020:                     uint32 val = (rt & 0xFF) << (24 - (rs & 3) * 8);

1021:                     uint32 mask = 0xFFFFFFFF ^ uint32(0xFF << (24 - (rs & 3) * 8));

1026:                     uint32 val = (rt & 0xFFFF) << (16 - (rs & 2) * 8);

1027:                     uint32 mask = 0xFFFFFFFF ^ uint32(0xFFFF << (16 - (rs & 2) * 8));

1032:                     uint32 val = rt >> ((rs & 3) * 8);

1033:                     uint32 mask = uint32(0xFFFFFFFF) >> ((rs & 3) * 8);

1042:                     uint32 val = rt << (24 - (rs & 3) * 8);

1043:                     uint32 mask = uint32(0xFFFFFFFF) << (24 - (rs & 3) * 8);

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/MIPS.sol)

```solidity
File: src/cannon/PreimageOracle.sol

4: import { IPreimageOracle } from "./interfaces/IPreimageOracle.sol";

5: import { ISemver } from "src/universal/ISemver.sol";

6: import { PreimageKeyLib } from "./PreimageKeyLib.sol";

7: import { LibKeccak } from "@lib-keccak/LibKeccak.sol";

8: import "src/cannon/libraries/CannonErrors.sol";

9: import "src/cannon/libraries/CannonTypes.sol";

29:     uint256 public constant MAX_LEAF_COUNT = 2 ** KECCAK_TREE_DEPTH - 1;

94:         for (uint256 height = 0; height < KECCAK_TREE_DEPTH - 1; height++) {

95:             zeroHashes[height + 1] = keccak256(abi.encodePacked(zeroHashes[height], zeroHashes[height]));

105:         require(preimagePartOk[_key][_offset], "pre-image must exist");

111:         if (_offset + 32 >= length + 8) {

112:             datLen_ = length + 8 - _offset;

134:         if (_partOffset > _size + 8 || _size > 32) {

185:             let h := keccak256(ptr, size) // compute preimage keccak256 hash

225:                     gas(), // Forward all available gas

226:                     0x02, // Address of SHA-256 precompile

227:                     ptr, // Start of input data in memory

228:                     size, // Size of input data

229:                     0, // Store output in scratch memory

230:                     0x20 // Output is always 32 bytes

234:             let h := mload(0) // get return data

288:                     gas(), // forward all gas

289:                     0x0A, // point evaluation precompile address

290:                     ptr, // input ptr

291:                     0xC0, // input size = 192 bytes

292:                     0x00, // output ptr

293:                     0x00 // output size

356:                     gas(), // forward all gas

358:                     add(20, ptr), // input ptr

360:                     0x0, // Unused as we don't copy anything

361:                     0x00 // don't copy anything

425:         if (_partOffset >= _claimedSize + 8) revert PartOffsetOOB();

539:             uint32(_input.length + metaData.bytesProcessed())

592:         if (_preState.index + 1 != _postState.index) revert StatesNotContiguous();

657:         if (block.timestamp - metaData.timestamp() <= CHALLENGE_PERIOD) revert ActiveProposal();

672:         if (_preState.index + 1 != _postState.index || _postState.index != metaData.blocksProcessed() - 1) {

698:         for (uint256 height = 0; height < KECCAK_TREE_DEPTH; height++) {

735:         } else if (offset >= 8 && (offset = offset - 8) >= currentSize && offset < currentSize + _input.length) {

736:             uint256 relativeOffset = offset - currentSize;

742:             if (relativeOffset + 32 >= _input.length && !_finalize) revert PartOffsetOOB();

768:             function hashTwo(a, b) -> hash {

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

```solidity
File: src/dispute/DisputeGameFactory.sol

4: import { LibClone } from "@solady/utils/LibClone.sol";

5: import { OwnableUpgradeable } from "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";

6: import { ISemver } from "src/universal/ISemver.sol";

8: import { IDisputeGame } from "src/dispute/interfaces/IDisputeGame.sol";

9: import { IDisputeGameFactory } from "src/dispute/interfaces/IDisputeGameFactory.sol";

11: import "src/dispute/lib/Types.sol";

12: import "src/dispute/lib/Errors.sol";

103:         bytes32 parentHash = blockhash(block.number - 1);

182:                 games_[games_.length - 1] = GameSearchResult({

193:                 i--;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

4: import { FixedPointMathLib } from "@solady/utils/FixedPointMathLib.sol";

6: import { IDelayedWETH } from "src/dispute/interfaces/IDelayedWETH.sol";

7: import { IDisputeGame } from "src/dispute/interfaces/IDisputeGame.sol";

8: import { IFaultDisputeGame } from "src/dispute/interfaces/IFaultDisputeGame.sol";

9: import { IInitializable } from "src/dispute/interfaces/IInitializable.sol";

10: import { IBigStepper, IPreimageOracle } from "src/dispute/interfaces/IBigStepper.sol";

11: import { IAnchorStateRegistry } from "src/dispute/interfaces/IAnchorStateRegistry.sol";

13: import { Clone } from "@solady/utils/Clone.sol";

14: import { Types } from "src/libraries/Types.sol";

15: import { ISemver } from "src/universal/ISemver.sol";

17: import { Types } from "src/libraries/Types.sol";

18: import { Hashing } from "src/libraries/Hashing.sol";

19: import { RLPReader } from "src/libraries/rlp/RLPReader.sol";

20: import "src/dispute/lib/Types.sol";

21: import "src/dispute/lib/Errors.sol";

138:         if (_maxGameDepth > LibPosition.MAX_POSITION_BITLEN - 1) revert MaxDepthTooLarge();

255:         if (stepPos.depth() != MAX_GAME_DEPTH + 1) revert InvalidParent();

269:             preStateClaim = (stepPos.indexAtDepth() % (1 << (MAX_GAME_DEPTH - SPLIT_DEPTH))) == 0

271:                 : _findTraceAncestor(Position.wrap(parentPos.raw() - 1), parent.parentIndex, false).claim;

278:             postState = _findTraceAncestor(Position.wrap(parentPos.raw() + 1), parent.parentIndex, false);

303:         bool parentPostAgree = (parentPos.depth() - postState.position.depth()) % 2 == 0;

339:         if ((_challengeIndex == 0 || nextPositionDepth == SPLIT_DEPTH + 2) && !_isAttack) {

355:         if (nextPositionDepth == SPLIT_DEPTH + 1) {

377:         if (nextDuration.raw() > MAX_CLOCK_DURATION.raw() - CLOCK_EXTENSION.raw()) {

380:                 nextPositionDepth == SPLIT_DEPTH - 1 ? CLOCK_EXTENSION.raw() * 2 : CLOCK_EXTENSION.raw();

381:             nextDuration = Duration.wrap(MAX_CLOCK_DURATION.raw() - extensionPeriod);

409:         subgames[_challengeIndex].push(claimData.length - 1);

453:             uint256 l2Number = startingOutputRoot.l2BlockNumber + disputedPos.traceIndex(SPLIT_DEPTH) + 1;

470:         numRemainingChildren_ = challengeIndicesLen - checkpoint.subgameIndex;

605:         uint256 lastToResolve = checkpoint.subgameIndex + _numToResolve;

607:         for (uint256 i = checkpoint.subgameIndex; i < finalCursor; i++) {

721:         uint256 a = highGasCharged / baseGasCharged;

723:         uint256 c = MAX_GAME_DEPTH * FixedPointMathLib.WAD;

727:         uint256 lnA = uint256(FixedPointMathLib.lnWad(int256(a * FixedPointMathLib.WAD)));

738:         int256 rawGas = FixedPointMathLib.powWad(base, int256(depth * FixedPointMathLib.WAD));

742:         requiredBond_ = assumedBaseFee * requiredGas;

784:             uint64(parentClock.duration().raw() + (block.timestamp - subgameRootClaim.clock.timestamp().raw()));

854:         credit[_recipient] += bond;

879:         Position disputedLeafPos = Position.wrap(_parentPos.raw() + 1);

957:             if (currentDepth == SPLIT_DEPTH + 1) execRootClaim = claim;

979:                 ClaimData storage starting = _findTraceAncestor(Position.wrap(outputPos.raw() - 1), claimIdx, true);

986:             ClaimData storage disputed = _findTraceAncestor(Position.wrap(outputPos.raw() + 1), claimIdx, true);

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="GAS-6"></a>[GAS-6] Use Custom Errors instead of Revert Strings to save Gas
Custom errors are available from solidity version 0.8.4. Custom errors save [**~50 gas**](https://gist.github.com/IllIllI000/ad1bd0d29a0101b25e57c293b4b0c746) each time they're hit by [avoiding having to allocate and store the revert string](https://blog.soliditylang.org/2021/04/21/custom-errors/#errors-in-depth). Not defining the strings also save deployment gas

Additionally, custom errors can be used inside and outside of contracts (including interfaces and libraries).

Source: <https://blog.soliditylang.org/2021/04/21/custom-errors/>:

> Starting from [Solidity v0.8.4](https://github.com/ethereum/solidity/releases/tag/v0.8.4), there is a convenient and gas-efficient way to explain to users why an operation failed through the use of custom errors. Until now, you could already use strings to give more information about failures (e.g., `revert("Insufficient funds.");`), but they are rather expensive, especially when it comes to deploy cost, and it is difficult to use dynamic information in them.

Consider replacing **all revert strings** with custom errors in the solution, and particularly those that have multiple occurrences:

*Instances (10)*:
```solidity
File: src/cannon/MIPS.sol

330:                 revert("branch in delay slot");

426:                     revert("MIPS: division by zero");

436:                     revert("MIPS: division by zero");

469:                 revert("jump in delay slot");

501:             require(_storeReg < 32, "valid register");

529:             require(s >= (offset_ + 28 * 32), "check that there is enough calldata");

959:                     revert("invalid instruction");

1054:                     revert("invalid instruction");

1057:             revert("invalid instruction");

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/MIPS.sol)

```solidity
File: src/cannon/PreimageOracle.sol

105:         require(preimagePartOk[_key][_offset], "pre-image must exist");

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

### <a name="GAS-7"></a>[GAS-7] State variables only set in the constructor should be declared `immutable`
Variables only set in the constructor and never edited afterwards should be marked as immutable, as it would avoid the expensive storage-writing operation in the constructor (around **20 000 gas** per variable) and replace the expensive storage-reading operations (around **2100 gas** per reading) to a less expensive value reading (**3 gas**)

*Instances (13)*:
```solidity
File: src/cannon/MIPS.sol

65:         ORACLE = _oracle;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/MIPS.sol)

```solidity
File: src/cannon/PreimageOracle.sol

90:         MIN_LPP_SIZE_BYTES = _minProposalSize;

91:         CHALLENGE_PERIOD = _challengePeriod;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

144:         GAME_TYPE = _gameType;

145:         ABSOLUTE_PRESTATE = _absolutePrestate;

146:         MAX_GAME_DEPTH = _maxGameDepth;

147:         SPLIT_DEPTH = _splitDepth;

148:         CLOCK_EXTENSION = _clockExtension;

149:         MAX_CLOCK_DURATION = _maxClockDuration;

150:         VM = _vm;

151:         WETH = _weth;

152:         ANCHOR_STATE_REGISTRY = _anchorStateRegistry;

153:         L2_CHAIN_ID = _l2ChainId;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="GAS-8"></a>[GAS-8] Functions guaranteed to revert when called by normal users can be marked `payable`
If a function modifier such as `onlyOwner` is used, the function will revert if a normal user tries to pay the function. Marking the function as `payable` will lower the gas cost for legitimate callers because the compiler will not include checks for whether a payment was provided.

*Instances (2)*:
```solidity
File: src/dispute/DisputeGameFactory.sol

199:     function setImplementation(GameType _gameType, IDisputeGame _impl) external onlyOwner {

205:     function setInitBond(GameType _gameType, uint256 _initBond) external onlyOwner {

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol)

### <a name="GAS-9"></a>[GAS-9] `++i` costs less gas compared to `i++` or `i += 1` (same for `--i` vs `i--` or `i -= 1`)
Pre-increments and pre-decrements are cheaper.

For a `uint256 i` variable, the following is true with the Optimizer enabled at 10k:

**Increment:**

- `i += 1` is the most expensive form
- `i++` costs 6 gas less than `i += 1`
- `++i` costs 5 gas less than `i++` (11 gas less than `i += 1`)

**Decrement:**

- `i -= 1` is the most expensive form
- `i--` costs 11 gas less than `i -= 1`
- `--i` costs 5 gas less than `i--` (16 gas less than `i -= 1`)

Note that post-increments (or post-decrements) return the old value before incrementing or decrementing, hence the name *post-increment*:

```solidity
uint i = 1;  
uint j = 2;
require(j == i++, "This will be false as i is incremented after the comparison");
```
  
However, pre-increments (or pre-decrements) return the new value:
  
```solidity
uint i = 1;  
uint j = 2;
require(j == ++i, "This will be true as i is incremented before the comparison");
```

In the pre-increment case, the compiler has to create a temporary variable (when used) for returning `1` instead of `2`.

Consider using pre-increments and pre-decrements where they are relevant (meaning: not where post-increments/decrements logic are relevant).

*Saves 5 gas per instance*

*Instances (5)*:
```solidity
File: src/cannon/MIPS.sol

976:                             i++;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/MIPS.sol)

```solidity
File: src/cannon/PreimageOracle.sol

94:         for (uint256 height = 0; height < KECCAK_TREE_DEPTH - 1; height++) {

698:         for (uint256 height = 0; height < KECCAK_TREE_DEPTH; height++) {

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

```solidity
File: src/dispute/DisputeGameFactory.sol

193:                 i--;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

607:         for (uint256 i = checkpoint.subgameIndex; i < finalCursor; i++) {

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="GAS-10"></a>[GAS-10] Using `private` rather than `public` for constants, saves gas
If needed, the values can be read from the verified contract source code, or if there are multiple values there can be a single getter function that [returns a tuple](https://github.com/code-423n4/2022-08-frax/blob/90f55a9ce4e25bceed3a74290b854341d8de6afa/src/contracts/FraxlendPair.sol#L156-L178) of the values of all currently-public constants. Saves **3406-3606 gas** in deployment gas due to the compiler not having to create non-payable getter functions for deployment calldata, not having to store the bytes of the value outside of where it's used, and not adding another entry to the method ID table

*Instances (8)*:
```solidity
File: src/cannon/MIPS.sol

43:     uint32 public constant BRK_START = 0x40000000;

47:     string public constant version = "1.0.1";

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/MIPS.sol)

```solidity
File: src/cannon/PreimageOracle.sol

25:     uint256 public constant MIN_BOND_SIZE = 0.25 ether;

27:     uint256 public constant KECCAK_TREE_DEPTH = 16;

29:     uint256 public constant MAX_LEAF_COUNT = 2 ** KECCAK_TREE_DEPTH - 1;

33:     string public constant version = "1.0.0";

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

```solidity
File: src/dispute/DisputeGameFactory.sol

25:     string public constant version = "1.0.0";

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

73:     string public constant version = "1.2.0";

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="GAS-11"></a>[GAS-11] Use shift right/left instead of division/multiplication if possible
While the `DIV` / `MUL` opcode uses 5 gas, the `SHR` / `SHL` opcode only uses 3 gas. Furthermore, beware that Solidity's division operation also includes a division-by-0 prevention which is bypassed using shifting. Eventually, overflow checks are never performed for shift operations as they are done for arithmetic operations. Instead, the result is always truncated, so the calculation can be unchecked in Solidity version `0.8+`
- Use `>> 1` instead of `/ 2`
- Use `>> 2` instead of `/ 4`
- Use `<< 3` instead of `* 8`
- ...
- Use `>> 5` instead of `/ 2^5 == / 32`
- Use `<< 6` instead of `* 2^6 == * 64`

TL;DR:
- Shifting left by N is like multiplying by 2^N (Each bits to the left is an increased power of 2)
- Shifting right by N is like dividing by 2^N (Each bits to the right is a decreased power of 2)

*Saves around 2 gas + 20 for unchecked per instance*

*Instances (19)*:
```solidity
File: src/cannon/MIPS.sol

524:             offset_ = 420 + (uint256(_proofIndex) * (28 * 32));

529:             require(s >= (offset_ + 28 * 32), "check that there is enough calldata");

988:                     return SE((mem >> (24 - (rs & 3) * 8)) & 0xFF, 8);

992:                     return SE((mem >> (16 - (rs & 2) * 8)) & 0xFFFF, 16);

996:                     uint32 val = mem << ((rs & 3) * 8);

997:                     uint32 mask = uint32(0xFFFFFFFF) << ((rs & 3) * 8);

1006:                     return (mem >> (24 - (rs & 3) * 8)) & 0xFF;

1010:                     return (mem >> (16 - (rs & 2) * 8)) & 0xFFFF;

1014:                     uint32 val = mem >> (24 - (rs & 3) * 8);

1015:                     uint32 mask = uint32(0xFFFFFFFF) >> (24 - (rs & 3) * 8);

1020:                     uint32 val = (rt & 0xFF) << (24 - (rs & 3) * 8);

1021:                     uint32 mask = 0xFFFFFFFF ^ uint32(0xFF << (24 - (rs & 3) * 8));

1026:                     uint32 val = (rt & 0xFFFF) << (16 - (rs & 2) * 8);

1027:                     uint32 mask = 0xFFFFFFFF ^ uint32(0xFFFF << (16 - (rs & 2) * 8));

1032:                     uint32 val = rt >> ((rs & 3) * 8);

1033:                     uint32 mask = uint32(0xFFFFFFFF) >> ((rs & 3) * 8);

1042:                     uint32 val = rt << (24 - (rs & 3) * 8);

1043:                     uint32 mask = uint32(0xFFFFFFFF) << (24 - (rs & 3) * 8);

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/MIPS.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

380:                 nextPositionDepth == SPLIT_DEPTH - 1 ? CLOCK_EXTENSION.raw() * 2 : CLOCK_EXTENSION.raw();

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="GAS-12"></a>[GAS-12] `uint256` to `bool` `mapping`: Utilizing Bitmaps to dramatically save on Gas
https://soliditydeveloper.com/bitmaps

https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/structs/BitMaps.sol

- [BitMaps.sol#L5-L16](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/structs/BitMaps.sol#L5-L16):

```solidity
/**
 * @dev Library for managing uint256 to bool mapping in a compact and efficient way, provided the keys are sequential.
 * Largely inspired by Uniswap's https://github.com/Uniswap/merkle-distributor/blob/master/contracts/MerkleDistributor.sol[merkle-distributor].
 *
 * BitMaps pack 256 booleans across each bit of a single 256-bit slot of `uint256` type.
 * Hence booleans corresponding to 256 _sequential_ indices would only consume a single slot,
 * unlike the regular `bool` which would consume an entire slot for a single value.
 *
 * This results in gas savings in two ways:
 *
 * - Setting a zero value to non-zero only once every 256 times
 * - Accessing the same warm slot for every 256 _sequential_ indices
 */
```

*Instances (2)*:
```solidity
File: src/cannon/PreimageOracle.sol

44:     mapping(bytes32 => mapping(uint256 => bool)) public preimagePartOk;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

107:     mapping(uint256 => bool) public resolvedSubgames;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="GAS-13"></a>[GAS-13] Increments/decrements can be unchecked in for-loops
In Solidity 0.8+, there's a default overflow check on unsigned integers. It's possible to uncheck this in for-loops and save some gas at each iteration, but at the cost of some code readability, as this uncheck cannot be made inline.

[ethereum/solidity#10695](https://github.com/ethereum/solidity/issues/10695)

The change would be:

```diff
- for (uint256 i; i < numIterations; i++) {
+ for (uint256 i; i < numIterations;) {
 // ...  
+   unchecked { ++i; }
}  
```

These save around **25 gas saved** per instance.

The same can be applied with decrements (which should use `break` when `i == 0`).

The risk of overflow is non-existent for `uint256`.

*Instances (3)*:
```solidity
File: src/cannon/PreimageOracle.sol

94:         for (uint256 height = 0; height < KECCAK_TREE_DEPTH - 1; height++) {

698:         for (uint256 height = 0; height < KECCAK_TREE_DEPTH; height++) {

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

607:         for (uint256 i = checkpoint.subgameIndex; i < finalCursor; i++) {

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="GAS-14"></a>[GAS-14] Use != 0 instead of > 0 for unsigned integer comparison

*Instances (2)*:
```solidity
File: src/cannon/MIPS.sol

344:                 shouldBranch = int32(_rs) > 0;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/MIPS.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

978:             if (outputPos.indexAtDepth() > 0) {

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="GAS-15"></a>[GAS-15] `internal` functions not called by the contract should be removed
If the functions are required by an interface, the contract should inherit from that interface and use the `override` keyword

*Instances (13)*:
```solidity
File: src/cannon/PreimageKeyLib.sol

12:     function localizeIdent(uint256 _ident, bytes32 _localContext) internal view returns (bytes32 key_) {

47:     function keccak256PreimageKey(bytes memory _preimage) internal pure returns (bytes32 key_) {

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageKeyLib.sol)

```solidity
File: src/cannon/libraries/CannonTypes.sol

32:             self_ := or(shl(160, _partOffset), and(_self, not(shl(160, U32_MASK))))

38:             self_ := or(shl(128, _claimedSize), and(_self, not(shl(128, U32_MASK))))

44:             self_ := or(shl(96, _blocksProcessed), and(_self, not(shl(96, U32_MASK))))

50:             self_ := or(shl(64, _bytesProcessed), and(_self, not(shl(64, U32_MASK))))

56:             self_ := or(_countered, and(_self, not(U64_MASK)))

66:     function partOffset(LPPMetaData _self) internal pure returns (uint64 partOffset_) {

72:     function claimedSize(LPPMetaData _self) internal pure returns (uint32 claimedSize_) {

78:     function blocksProcessed(LPPMetaData _self) internal pure returns (uint32 blocksProcessed_) {

84:     function bytesProcessed(LPPMetaData _self) internal pure returns (uint32 bytesProcessed_) {

90:     function countered(LPPMetaData _self) internal pure returns (bool countered_) {

96: 

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/libraries/CannonTypes.sol)

### <a name="GAS-16"></a>[GAS-16] WETH address definition can be use directly
WETH is a wrap Ether contract with a specific address in the Ethereum network, giving the option to define it may cause false recognition, it is healthier to define it directly.

    Advantages of defining a specific contract directly:
    
    It saves gas,
    Prevents incorrect argument definition,
    Prevents execution on a different chain and re-signature issues,
    WETH Address : 0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2

*Instances (1)*:
```solidity
File: src/dispute/FaultDisputeGame.sol

51:     IDelayedWETH internal immutable WETH;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)


## Non Critical Issues


| |Issue|Instances|
|-|:-|:-:|
| [NC-1](#NC-1) | Array indices should be referenced via `enum`s rather than via numeric literals | 10 |
| [NC-2](#NC-2) | Use `string.concat()` or `bytes.concat()` instead of `abi.encodePacked` | 3 |
| [NC-3](#NC-3) | Constants should be in CONSTANT_CASE | 4 |
| [NC-4](#NC-4) | `constant`s should be defined rather than using magic numbers | 154 |
| [NC-5](#NC-5) | Control structures do not follow the Solidity Style Guide | 77 |
| [NC-6](#NC-6) | Consider disabling `renounceOwnership()` | 1 |
| [NC-7](#NC-7) | Events that mark critical parameter changes should contain both the old and the new value | 2 |
| [NC-8](#NC-8) | Function ordering does not follow the Solidity style guide | 3 |
| [NC-9](#NC-9) | Functions should not be longer than 50 lines | 78 |
| [NC-10](#NC-10) | Change int to int256 | 3 |
| [NC-11](#NC-11) | Lack of checks in setters | 2 |
| [NC-12](#NC-12) | Incomplete NatSpec: `@param` is missing on actually documented functions | 5 |
| [NC-13](#NC-13) | Incomplete NatSpec: `@return` is missing on actually documented functions | 1 |
| [NC-14](#NC-14) | Use a `modifier` instead of a `require/if` statement for a special `msg.sender` actor | 2 |
| [NC-15](#NC-15) | Constant state variables defined more than once | 4 |
| [NC-16](#NC-16) | Consider using named mappings | 16 |
| [NC-17](#NC-17) | Adding a `return` statement when the function defines a named return variable, is redundant | 51 |
| [NC-18](#NC-18) | Take advantage of Custom Error's return value property | 64 |
| [NC-19](#NC-19) | Contract does not follow the Solidity style guide's suggested layout ordering | 2 |
| [NC-20](#NC-20) | Use Underscores for Number Literals (add an underscore every 3 digits) | 9 |
| [NC-21](#NC-21) | Internal and private variables and functions names should begin with an underscore | 26 |
| [NC-22](#NC-22) | Constants should be defined rather than using magic numbers | 41 |
| [NC-23](#NC-23) | `public` functions not called by the contract should be declared `external` instead | 2 |
| [NC-24](#NC-24) | Variables need not be initialized to zero | 6 |
### <a name="NC-1"></a>[NC-1] Array indices should be referenced via `enum`s rather than via numeric literals

*Instances (10)*:
```solidity
File: src/cannon/MIPS.sol

160:             uint32 syscall_no = state.registers[2];

165:             uint32 a0 = state.registers[4];

166:             uint32 a1 = state.registers[5];

167:             uint32 a2 = state.registers[6];

210:                     if (uint8(preimageKey[0]) == 1) {

302:             state.registers[2] = v0;

303:             state.registers[7] = v1;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/MIPS.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

546:         if (!resolvedSubgames[0]) revert OutOfOrderResolution();

549:         status_ = claimData[0].counteredBy == address(0) ? GameStatus.DEFENDER_WINS : GameStatus.CHALLENGER_WINS;

881:         uint8 vmStatus = uint8(_rootClaim.raw()[0]);

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="NC-2"></a>[NC-2] Use `string.concat()` or `bytes.concat()` instead of `abi.encodePacked`
Solidity version 0.8.4 introduces `bytes.concat()` (vs `abi.encodePacked(<bytes>,<bytes>)`)

Solidity version 0.8.12 introduces `string.concat()` (vs `abi.encodePacked(<str>,<str>), which catches concatenation errors (in the event of a `bytes` data mixed in the concatenation)`)

*Instances (3)*:
```solidity
File: src/cannon/PreimageOracle.sol

95:             zeroHashes[height + 1] = keccak256(abi.encodePacked(zeroHashes[height], zeroHashes[height]));

798:         leaf_ = keccak256(abi.encodePacked(_leaf.input, _leaf.index, _leaf.stateCommitment));

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

```solidity
File: src/dispute/DisputeGameFactory.sol

116:         proxy_ = IDisputeGame(address(impl).clone(abi.encodePacked(msg.sender, _rootClaim, parentHash, _extraData)));

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol)

### <a name="NC-3"></a>[NC-3] Constants should be in CONSTANT_CASE
For `constant` variable names, each word should use all capital letters, with underscores separating each word (CONSTANT_CASE)

*Instances (4)*:
```solidity
File: src/cannon/MIPS.sol

47:     string public constant version = "1.0.1";

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/MIPS.sol)

```solidity
File: src/cannon/PreimageOracle.sol

33:     string public constant version = "1.0.0";

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

```solidity
File: src/dispute/DisputeGameFactory.sol

25:     string public constant version = "1.0.0";

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

73:     string public constant version = "1.2.0";

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="NC-4"></a>[NC-4] `constant`s should be defined rather than using magic numbers
Even [assembly](https://github.com/code-423n4/2022-05-opensea-seaport/blob/9d7ce4d08bf3c3010304a0476a785c70c0e90ae7/contracts/lib/TokenTransferrer.sol#L35-L39) can benefit from using readable constants instead of hex/numeric literals

*Instances (154)*:
```solidity
File: src/cannon/MIPS.sol

91:                 fromOut := add(from, 32)

103:             from, to := copyMem(from, to, 32) // memRoot

104:             from, to := copyMem(from, to, 32) // preimageKey

105:             from, to := copyMem(from, to, 4) // preimageOffset

106:             from, to := copyMem(from, to, 4) // pc

107:             from, to := copyMem(from, to, 4) // nextPC

108:             from, to := copyMem(from, to, 4) // lo

109:             from, to := copyMem(from, to, 4) // hi

110:             from, to := copyMem(from, to, 4) // heap

115:             from, to := copyMem(from, to, 8) // step

116:             from := add(from, 32) // offset to registers

119:             for { let i := 0 } lt(i, 32) { i := add(i, 1) } { from, to := copyMem(from, to, 4) }

137:                 default { status := 2 }

140:             default { status := 3 }

170:             if (syscall_no == 4090) {

172:                 if (sz & 4095 != 0) {

174:                     sz += 4096 - (sz & 4095);

184:             else if (syscall_no == 4045) {

188:             else if (syscall_no == 4120) {

192:             else if (syscall_no == 4246) {

198:             else if (syscall_no == 4003) {

218:                         let alignment := and(a1, 3) // the read might not start at an aligned address

222:                         dat := shr(sub(256, mul(datLen, 8)), dat) // right-align data

223:                         dat := shl(mul(sub(sub(4, datLen), alignment), 8), dat) // position data to insert into memory

225:                         let mask := sub(shl(mul(sub(4, alignment), 8), 1), 1) // mask all bytes after start

226:                         let suffixMask := sub(shl(mul(sub(sub(4, alignment), datLen), 8), 1), 1) // mask of all bytes

248:             else if (syscall_no == 4004) {

262:                         let alignment := and(a1, 3) // the read might not start at an aligned address

265:                         key := shl(mul(a2, 8), key) // shift key, make space for new info

266:                         let mask := sub(shl(mul(a2, 8), 1), 1) // mask for extracting value from memory

267:                         mem := and(shr(mul(sub(space, a2), 8), mem), mask) // align value to right, mask it

282:             else if (syscall_no == 4055) {

285:                 if (a1 == 3) {

307:             state.nextPC = state.nextPC + 4;

329:             if (state.nextPC != state.pc + 4) {

334:             if (_opcode == 4 || _opcode == 5) {

336:                 shouldBranch = (_rs == rt && _opcode == 4) || (_rs != rt && _opcode == 5);

339:             else if (_opcode == 6) {

343:             else if (_opcode == 7) {

349:                 uint32 rtv = ((_insn >> 16) & 0x1F);

367:                 state.nextPC = prevPC + 4 + (SE(_insn & 0xFFFF, 16) << 2);

369:                 state.nextPC = state.nextPC + 4;

412:                 state.hi = uint32(acc >> 32);

418:                 state.hi = uint32(acc >> 32);

449:             state.nextPC = state.nextPC + 4;

468:             if (state.nextPC != state.pc + 4) {

479:                 state.registers[_linkReg] = prevPC + 8;

501:             require(_storeReg < 32, "valid register");

510:             state.nextPC = state.nextPC + 4;

524:             offset_ = 420 + (uint256(_proofIndex) * (28 * 32));

529:             require(s >= (offset_ + 28 * 32), "check that there is enough calldata");

545:                 if and(_addr, 3) { revert(0, 0) }

549:                 offset := add(offset, 32)

555:                     h := keccak256(0, 64)

562:                 for { let i := 0 } lt(i, 27) { i := add(i, 1) } {

564:                     offset := add(offset, 32)

576:                     revert(0, 32)

580:                 let shamt := shl(3, sub(sub(32, 4), and(_addr, 31)))

599:                 if and(_addr, 3) { revert(0, 0) }

603:                 let shamt := shl(3, sub(sub(32, 4), and(_addr, 31)))

607:                 offset := add(offset, 32)

613:                     h := keccak256(0, 64)

620:                 for { let i := 0 } lt(i, 27) { i := add(i, 1) } {

622:                     offset := add(offset, 32)

650:                 if iszero(eq(mload(0x40), shl(5, 48))) {

654:                 if iszero(eq(_stateData.offset, 132)) {

658:                 if iszero(eq(_proof.offset, 420)) {

668:                     memOffsetOut := add(memOffset, 32)

674:                 c, m := putField(c, m, 32) // memRoot

675:                 c, m := putField(c, m, 32) // preimageKey

676:                 c, m := putField(c, m, 4) // preimageOffset

677:                 c, m := putField(c, m, 4) // pc

678:                 c, m := putField(c, m, 4) // nextPC

679:                 c, m := putField(c, m, 4) // lo

680:                 c, m := putField(c, m, 4) // hi

681:                 c, m := putField(c, m, 4) // heap

684:                 c, m := putField(c, m, 8) // step

687:                 mstore(m, add(m, 32)) // offset to registers

688:                 m := add(m, 32)

689:                 for { let i := 0 } lt(i, 32) { i := add(i, 1) } { c, m := putField(c, m, 4) }

701:             uint32 opcode = insn >> 26; // 6-bits

704:             if (opcode == 2 || opcode == 3) {

706:                 uint32 target = (state.nextPC & 0xF0000000) | (insn & 0x03FFFFFF) << 2;

707:                 return handleJump(opcode == 2 ? 0 : 31, target);

713:             uint32 rtReg = (insn >> 16) & 0x1F;

716:             rs = state.registers[(insn >> 21) & 0x1F];

722:                 rdReg = (insn >> 11) & 0x1F;

731:                     rt = SE(insn & 0xFFFF, 16);

741:             if ((opcode >= 4 && opcode < 8) || opcode == 1) {

751:                 rs += SE(insn & 0xFFFF, 16);

766:             if (opcode == 0 && func >= 8 && func < 0x1c) {

767:                 if (func == 8 || func == 9) {

769:                     return handleJump(func == 8 ? 0 : rdReg, rs);

811:             uint32 opcode = insn >> 26; // 6-bits

813:             if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {

836:                     return rt << ((insn >> 6) & 0x1F);

840:                     return rt >> ((insn >> 6) & 0x1F);

844:                     uint32 shamt = (insn >> 6) & 0x1F;

845:                     return SE(rt >> shamt, 32 - shamt);

857:                     return SE(rt >> rs, 32 - rs);

984:                     return rt << 16;

988:                     return SE((mem >> (24 - (rs & 3) * 8)) & 0xFF, 8);

992:                     return SE((mem >> (16 - (rs & 2) * 8)) & 0xFFFF, 16);

996:                     uint32 val = mem << ((rs & 3) * 8);

997:                     uint32 mask = uint32(0xFFFFFFFF) << ((rs & 3) * 8);

1006:                     return (mem >> (24 - (rs & 3) * 8)) & 0xFF;

1010:                     return (mem >> (16 - (rs & 2) * 8)) & 0xFFFF;

1014:                     uint32 val = mem >> (24 - (rs & 3) * 8);

1015:                     uint32 mask = uint32(0xFFFFFFFF) >> (24 - (rs & 3) * 8);

1020:                     uint32 val = (rt & 0xFF) << (24 - (rs & 3) * 8);

1021:                     uint32 mask = 0xFFFFFFFF ^ uint32(0xFF << (24 - (rs & 3) * 8));

1026:                     uint32 val = (rt & 0xFFFF) << (16 - (rs & 2) * 8);

1027:                     uint32 mask = 0xFFFFFFFF ^ uint32(0xFFFF << (16 - (rs & 2) * 8));

1032:                     uint32 val = rt >> ((rs & 3) * 8);

1033:                     uint32 mask = uint32(0xFFFFFFFF) >> ((rs & 3) * 8);

1042:                     uint32 val = rt << (24 - (rs & 3) * 8);

1043:                     uint32 mask = uint32(0xFFFFFFFF) << (24 - (rs & 3) * 8);

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/MIPS.sol)

```solidity
File: src/cannon/PreimageKeyLib.sol

56:             key_ := or(and(h, not(shl(248, 0xFF))), shl(248, 2))

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageKeyLib.sol)

```solidity
File: src/cannon/PreimageOracle.sol

109:         datLen_ = 32;

111:         if (_offset + 32 >= length + 8) {

112:             datLen_ = length + 8 - _offset;

134:         if (_partOffset > _size + 8 || _size > 32) {

169:             if iszero(lt(_partOffset, add(size, 8))) {

204:             if iszero(lt(_partOffset, add(size, 8))) {

208:                 revert(0x1c, 4)

215:             ptr := add(ptr, 8)

220:             part := mload(add(sub(ptr, 8), _partOffset))

236:             key := or(and(h, not(shl(248, 0xFF))), shl(248, 4))

331:         preimageLengths[key] = 32;

346:             calldatacopy(ptr, 48, 20)

366:             if iszero(lt(_partOffset, add(size, 8))) {

370:                 revert(0x1c, 4)

425:         if (_partOffset >= _claimedSize + 8) revert PartOffsetOOB();

488:             if or(mod(inputLen, 136), iszero(eq(_stateCommitments.length, div(inputLen, 136)))) {

498:             for { let i := 0 } lt(i, inputLen) { i := add(i, 136) } {

506:                 mstore(add(hashBuf, 136), blocksProcessed)

507:                 mstore(add(hashBuf, 168), calldataload(add(_stateCommitments.offset, shl(0x05, div(i, 136)))))

727:         if (offset < 8 && currentSize == 0) {

735:         } else if (offset >= 8 && (offset = offset - 8) >= currentSize && offset < currentSize + _input.length) {

742:             if (relativeOffset + 32 >= _input.length && !_finalize) revert PartOffsetOOB();

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

285:         if (keccak256(_stateData) << 8 != preStateClaim.raw() << 8) revert InvalidPrestate();

303:         bool parentPostAgree = (parentPos.depth() - postState.position.depth()) % 2 == 0;

339:         if ((_challengeIndex == 0 || nextPositionDepth == SPLIT_DEPTH + 2) && !_isAttack) {

380:                 nextPositionDepth == SPLIT_DEPTH - 1 ? CLOCK_EXTENSION.raw() * 2 : CLOCK_EXTENSION.raw();

440:             oracle.loadLocalData(_ident, uuid.raw(), l1Head().raw(), 32, _partOffset);

443:             oracle.loadLocalData(_ident, uuid.raw(), starting.raw(), 32, _partOffset);

446:             oracle.loadLocalData(_ident, uuid.raw(), disputed.raw(), 32, _partOffset);

455:             oracle.loadLocalData(_ident, uuid.raw(), bytes32(l2Number << 0xC0), 8, _partOffset);

458:             oracle.loadLocalData(_ident, uuid.raw(), bytes32(L2_CHAIN_ID << 0xC0), 8, _partOffset);

516:         if (rawBlockNumber.length > 32) revert InvalidHeaderRLP();

707:         uint256 assumedBaseFee = 200 gwei;

708:         uint256 baseGasCharged = 400_000;

709:         uint256 highGasCharged = 300_000_000;

883:         if (_isAttack || disputed.position.depth() % 2 == SPLIT_DEPTH % 2) {

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="NC-5"></a>[NC-5] Control structures do not follow the Solidity Style Guide
See the [control structures](https://docs.soliditylang.org/en/latest/style-guide.html#control-structures) section of the Solidity Style Guide

*Instances (77)*:
```solidity
File: src/cannon/MIPS.sol

220:                         if lt(space, datLen) { datLen := space } // if less space than data, shorten data

221:                         if lt(a2, datLen) { datLen := a2 } // if requested to read less, read less

264:                         if lt(space, a2) { a2 := space } // if less space than data, shorten data

265:                         key := shl(mul(a2, 8), key) // shift key, make space for new info

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/MIPS.sol)

```solidity
File: src/cannon/PreimageOracle.sol

419:         if (msg.value < MIN_BOND_SIZE) revert InsufficientBond();

422:         if (msg.sender != tx.origin) revert NotEOA();

425:         if (_partOffset >= _claimedSize + 8) revert PartOffsetOOB();

428:         if (_claimedSize < MIN_LPP_SIZE_BYTES) revert InvalidInputSize();

465:         if (msg.sender != tx.origin) revert NotEOA();

468:         if (metaData.claimedSize() == 0) revert NotInitialized();

471:         if (metaData.timestamp() != 0) revert AlreadyFinalized();

475:         if (blocksProcessed != _inputStartBlock) revert WrongStartingBlock();

535:         if (blocksProcessed > MAX_LEAF_COUNT) revert TreeSizeOverflow();

547:             if (metaData.bytesProcessed() != metaData.claimedSize()) revert InvalidInputSize();

581:         if (

583:                 _verify(_preStateProof, root, _preState.index, _hashLeaf(_preState))

584:                     && _verify(_postStateProof, root, _postState.index, _hashLeaf(_postState))

589:         if (keccak256(abi.encode(_stateMatrix)) != _preState.stateCommitment) revert InvalidPreimage();

592:         if (_preState.index + 1 != _postState.index) revert StatesNotContiguous();

599:         if (keccak256(abi.encode(_stateMatrix)) == _postState.stateCommitment) revert PostStateMatches();

619:         if (!_verify(_postStateProof, root, _postState.index, _hashLeaf(_postState))) revert InvalidProof();

622:         if (_postState.index != 0) revert StatesNotContiguous();

630:         if (keccak256(abi.encode(stateMatrix)) == _postState.stateCommitment) revert PostStateMatches();

654:         if (metaData.countered()) revert BadProposal();

657:         if (block.timestamp - metaData.timestamp() <= CHALLENGE_PERIOD) revert ActiveProposal();

661:         if (

663:                 _verify(_preStateProof, root, _preState.index, _hashLeaf(_preState))

664:                     && _verify(_postStateProof, root, _postState.index, _hashLeaf(_postState))

669:         if (keccak256(abi.encode(_stateMatrix)) != _preState.stateCommitment) revert InvalidPreimage();

742:             if (relativeOffset + 32 >= _input.length && !_finalize) revert PartOffsetOOB();

756:     function _verify(

793:         if (!success) revert BondTransferFailed();

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

```solidity
File: src/dispute/DisputeGameFactory.sol

97:         if (address(impl) == address(0)) revert NoImplementation(_gameType);

100:         if (msg.value != initBonds[_gameType]) revert IncorrectBondAmount();

123:         if (GameId.unwrap(_disputeGames[uuid]) != bytes32(0)) revert GameAlreadyExists(uuid);

158:         if (_start >= _disputeGameList.length || _n == 0) return games_;

189:                 if (games_.length >= _n) break;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

8: import { IFaultDisputeGame } from "src/dispute/interfaces/IFaultDisputeGame.sol";

138:         if (_maxGameDepth > LibPosition.MAX_POSITION_BITLEN - 1) revert MaxDepthTooLarge();

140:         if (_splitDepth >= _maxGameDepth) revert InvalidSplitDepth();

142:         if (_clockExtension.raw() > _maxClockDuration.raw()) revert InvalidClockExtension();

170:         if (initialized) revert AlreadyInitialized();

176:         if (root.raw() == bytes32(0)) revert AnchorRootNotFound();

204:         if (l2BlockNumber() <= rootBlockNumber) revert UnexpectedRootClaim(rootClaim());

244:         if (status != GameStatus.IN_PROGRESS) revert GameNotInProgress();

255:         if (stepPos.depth() != MAX_GAME_DEPTH + 1) revert InvalidParent();

285:         if (keccak256(_stateData) << 8 != preStateClaim.raw() << 8) revert InvalidPrestate();

304:         if (parentPostAgree == validStep) revert ValidStep();

307:         if (parent.counteredBy != address(0)) revert DuplicateStep();

321:         if (status != GameStatus.IN_PROGRESS) revert GameNotInProgress();

327:         if (Claim.unwrap(parent.claim) != Claim.unwrap(_disputed)) revert InvalidDisputedClaimIndex();

345:         if (l2BlockNumberChallenged && _challengeIndex == 0) revert L2BlockNumberChallenged();

351:         if (nextPositionDepth > MAX_GAME_DEPTH) revert GameDepthExceeded();

356:             _verifyExecBisectionRoot(_claim, _challengeIndex, parentPos, _isAttack);

360:         if (getRequiredBond(nextPosition) != msg.value) revert IncorrectBondAmount();

369:         if (nextDuration.raw() == MAX_CLOCK_DURATION.raw()) revert ClockTimeExceeded();

391:         if (claims[claimHash]) revert ClaimAlreadyExists();

431:         if (status != GameStatus.IN_PROGRESS) revert GameNotInProgress();

499:         if (status != GameStatus.IN_PROGRESS) revert GameNotInProgress();

502:         if (l2BlockNumberChallenged) revert L2BlockNumberChallenged();

505:         if (Hashing.hashOutputRootProof(_outputRootProof) != rootClaim().raw()) revert InvalidOutputRootProof();

508:         if (keccak256(_headerRLP) != _outputRootProof.latestBlockhash) revert InvalidHeaderRLP();

516:         if (rawBlockNumber.length > 32) revert InvalidHeaderRLP();

528:         if (blockNumber == l2BlockNumber()) revert BlockNumberMatches();

543:         if (status != GameStatus.IN_PROGRESS) revert GameNotInProgress();

546:         if (!resolvedSubgames[0]) revert OutOfOrderResolution();

562:         if (status != GameStatus.IN_PROGRESS) revert GameNotInProgress();

570:         if (challengeClockDuration.raw() < MAX_CLOCK_DURATION.raw()) revert ClockNotExpired();

573:         if (resolvedSubgames[_claimIndex]) revert ClaimAlreadyResolved();

601:             if (_numToResolve == 0) _numToResolve = challengeIndicesLen;

611:             if (!resolvedSubgames[challengeIndex]) revert OutOfOrderResolution();

704:         if (depth > MAX_GAME_DEPTH) revert GameDepthExceeded();

753:         if (recipientCredit == 0) revert NoCreditToClaim();

760:         if (!success) revert BondTransferFailed();

863:     function _verifyExecBisectionRoot(

941:         if (claim.position.depth() <= SPLIT_DEPTH) revert ClaimAboveSplit();

957:             if (currentDepth == SPLIT_DEPTH + 1) execRootClaim = claim;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="NC-6"></a>[NC-6] Consider disabling `renounceOwnership()`
If the plan for your project does not include eventually giving up all ownership control, consider overwriting OpenZeppelin's `Ownable`'s `renounceOwnership()` function in order to disable it.

*Instances (1)*:
```solidity
File: src/dispute/DisputeGameFactory.sol

19: contract DisputeGameFactory is OwnableUpgradeable, IDisputeGameFactory, ISemver {

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol)

### <a name="NC-7"></a>[NC-7] Events that mark critical parameter changes should contain both the old and the new value
This should especially be done if the new value is not required to be different from the old value

*Instances (2)*:
```solidity
File: src/dispute/DisputeGameFactory.sol

206:         initBonds[_gameType] = _initBond;
             emit InitBondUpdated(_gameType, _initBond);
         }
     }
     

210: 

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol)

### <a name="NC-8"></a>[NC-8] Function ordering does not follow the Solidity style guide
According to the [Solidity style guide](https://docs.soliditylang.org/en/v0.8.17/style-guide.html#order-of-functions), functions should be laid out in the following order :`constructor()`, `receive()`, `fallback()`, `external`, `public`, `internal`, `private`, but the cases below do not follow this pattern

*Instances (3)*:
```solidity
File: src/cannon/MIPS.sol

1: 
   Current order:
   external oracle
   internal SE
   internal outputState
   internal handleSyscall
   internal handleBranch
   internal handleHiLo
   internal handleJump
   internal handleRd
   internal proofOffset
   internal readMem
   internal writeMem
   public step
   internal execute
   
   Suggested order:
   external oracle
   public step
   internal SE
   internal outputState
   internal handleSyscall
   internal handleBranch
   internal handleHiLo
   internal handleJump
   internal handleRd
   internal proofOffset
   internal readMem
   internal writeMem
   internal execute

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/MIPS.sol)

```solidity
File: src/dispute/DisputeGameFactory.sol

1: 
   Current order:
   public initialize
   external gameCount
   external games
   external gameAtIndex
   external create
   public getGameUUID
   external findLatestGames
   external setImplementation
   external setInitBond
   
   Suggested order:
   external gameCount
   external games
   external gameAtIndex
   external create
   external findLatestGames
   external setImplementation
   external setInitBond
   public initialize
   public getGameUUID

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

1: 
   Current order:
   public initialize
   public step
   public move
   external attack
   external defend
   external addLocalData
   public getNumToResolve
   public l2BlockNumber
   external startingBlockNumber
   external startingRootHash
   external challengeRootL2Block
   external resolve
   external resolveClaim
   public gameType
   public gameCreator
   public rootClaim
   public l1Head
   public extraData
   external gameData
   public getRequiredBond
   external claimCredit
   public getChallengerDuration
   external claimDataLen
   external absolutePrestate
   external maxGameDepth
   external splitDepth
   external maxClockDuration
   external clockExtension
   external vm
   external weth
   external anchorStateRegistry
   external l2ChainId
   internal _distributeBond
   internal _verifyExecBisectionRoot
   internal _findTraceAncestor
   internal _findStartingAndDisputedOutputs
   internal _findLocalContext
   internal _computeLocalContext
   
   Suggested order:
   external attack
   external defend
   external addLocalData
   external startingBlockNumber
   external startingRootHash
   external challengeRootL2Block
   external resolve
   external resolveClaim
   external gameData
   external claimCredit
   external claimDataLen
   external absolutePrestate
   external maxGameDepth
   external splitDepth
   external maxClockDuration
   external clockExtension
   external vm
   external weth
   external anchorStateRegistry
   external l2ChainId
   public initialize
   public step
   public move
   public getNumToResolve
   public l2BlockNumber
   public gameType
   public gameCreator
   public rootClaim
   public l1Head
   public extraData
   public getRequiredBond
   public getChallengerDuration
   internal _distributeBond
   internal _verifyExecBisectionRoot
   internal _findTraceAncestor
   internal _findStartingAndDisputedOutputs
   internal _findLocalContext
   internal _computeLocalContext

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="NC-9"></a>[NC-9] Functions should not be longer than 50 lines
Overly complex code can make understanding functionality more difficult, try to further modularize your code to ensure readability 

*Instances (78)*:
```solidity
File: src/cannon/MIPS.sol

70:     function oracle() external view returns (IPreimageOracle oracle_) {

75:     function SE(uint32 _dat, uint32 _idx) internal pure returns (uint32 out_) {

86:     function outputState() internal returns (bytes32 out_) {

89:             function copyMem(from, to, size) -> fromOut, toOut {

151:     function handleSyscall(bytes32 _localContext) internal returns (bytes32 out_) {

319:     function handleBranch(uint32 _opcode, uint32 _insn, uint32 _rtReg, uint32 _rs) internal returns (bytes32 out_) {

383:     function handleHiLo(uint32 _func, uint32 _rs, uint32 _rt, uint32 _storeReg) internal returns (bytes32 out_) {

460:     function handleJump(uint32 _linkReg, uint32 _dest) internal returns (bytes32 out_) {

492:     function handleRd(uint32 _storeReg, uint32 _val, bool _conditional) internal returns (bytes32 out_) {

520:     function proofOffset(uint8 _proofIndex) internal pure returns (uint256 offset_) {

538:     function readMem(uint32 _addr, uint8 _proofIndex) internal pure returns (uint32 out_) {

592:     function writeMem(uint32 _addr, uint8 _proofIndex, uint32 _val) internal pure {

640:     function step(bytes calldata _stateData, bytes calldata _proof, bytes32 _localContext) public returns (bytes32) {

663:                 function putField(callOffset, memOffset, size) -> callOffsetOut, memOffsetOut {

809:     function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/MIPS.sol)

```solidity
File: src/cannon/PreimageKeyLib.sol

12:     function localizeIdent(uint256 _ident, bytes32 _localContext) internal view returns (bytes32 key_) {

29:     function localize(bytes32 _key, bytes32 _localContext) internal view returns (bytes32 localizedKey_) {

47:     function keccak256PreimageKey(bytes memory _preimage) internal pure returns (bytes32 key_) {

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageKeyLib.sol)

```solidity
File: src/cannon/PreimageOracle.sol

104:     function readPreimage(bytes32 _key, uint256 _offset) external view returns (bytes32 dat_, uint256 datLen_) {

160:     function loadKeccak256PreimagePart(uint256 _partOffset, bytes calldata _preimage) external {

195:     function loadSha256PreimagePart(uint256 _partOffset, bytes calldata _preimage) external {

335:     function loadPrecompilePreimagePart(uint256 _partOffset, address _precompile, bytes calldata _input) external {

396:     function proposalCount() external view returns (uint256 count_) {

402:     function proposalBlocksLen(address _claimant, uint256 _uuid) external view returns (uint256 len_) {

407:     function challengePeriod() external view returns (uint256 challengePeriod_) {

412:     function minProposalSize() external view returns (uint256 minProposalSize_) {

417:     function initLPP(uint256 _uuid, uint32 _partOffset, uint32 _claimedSize) external payable {

696:     function getTreeRootLPP(address _owner, uint256 _uuid) public view returns (bytes32 treeRoot_) {

788:     function _payoutBond(address _claimant, uint256 _uuid, address _to) internal {

797:     function _hashLeaf(Leaf memory _leaf) internal pure returns (bytes32 leaf_) {

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

```solidity
File: src/cannon/libraries/CannonTypes.sol

24:     function setTimestamp(LPPMetaData _self, uint64 _timestamp) internal pure returns (LPPMetaData self_) {

30:     function setPartOffset(LPPMetaData _self, uint32 _partOffset) internal pure returns (LPPMetaData self_) {

36:     function setClaimedSize(LPPMetaData _self, uint32 _claimedSize) internal pure returns (LPPMetaData self_) {

42:     function setBlocksProcessed(LPPMetaData _self, uint32 _blocksProcessed) internal pure returns (LPPMetaData self_) {

48:     function setBytesProcessed(LPPMetaData _self, uint32 _bytesProcessed) internal pure returns (LPPMetaData self_) {

54:     function setCountered(LPPMetaData _self, bool _countered) internal pure returns (LPPMetaData self_) {

60:     function timestamp(LPPMetaData _self) internal pure returns (uint64 timestamp_) {

66:     function partOffset(LPPMetaData _self) internal pure returns (uint64 partOffset_) {

72:     function claimedSize(LPPMetaData _self) internal pure returns (uint32 claimedSize_) {

78:     function blocksProcessed(LPPMetaData _self) internal pure returns (uint32 blocksProcessed_) {

84:     function bytesProcessed(LPPMetaData _self) internal pure returns (uint32 bytesProcessed_) {

90:     function countered(LPPMetaData _self) internal pure returns (bool countered_) {

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/libraries/CannonTypes.sol)

```solidity
File: src/dispute/DisputeGameFactory.sol

48:     function initialize(address _owner) public initializer {

54:     function gameCount() external view returns (uint256 gameCount_) {

199:     function setImplementation(GameType _gameType, IDisputeGame _impl) external onlyOwner {

205:     function setInitBond(GameType _gameType, uint256 _initBond) external onlyOwner {

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

319:     function move(Claim _disputed, uint256 _challengeIndex, Claim _claim, bool _isAttack) public payable virtual {

419:     function attack(Claim _disputed, uint256 _parentIndex, Claim _claim) external payable {

424:     function defend(Claim _disputed, uint256 _parentIndex, Claim _claim) external payable {

429:     function addLocalData(uint256 _ident, uint256 _execLeafIdx, uint256 _partOffset) external {

465:     function getNumToResolve(uint256 _claimIndex) public view returns (uint256 numRemainingChildren_) {

474:     function l2BlockNumber() public pure returns (uint256 l2BlockNumber_) {

479:     function startingBlockNumber() external view returns (uint256 startingBlockNumber_) {

484:     function startingRootHash() external view returns (Hash startingRootHash_) {

541:     function resolve() external returns (GameStatus status_) {

560:     function resolveClaim(uint256 _claimIndex, uint256 _numToResolve) external {

662:     function gameType() public view override returns (GameType gameType_) {

667:     function gameCreator() public pure returns (address creator_) {

672:     function rootClaim() public pure returns (Claim rootClaim_) {

677:     function l1Head() public pure returns (Hash l1Head_) {

682:     function extraData() public pure returns (bytes memory extraData_) {

689:     function gameData() external view returns (GameType gameType_, Claim rootClaim_, bytes memory extraData_) {

702:     function getRequiredBond(Position _position) public view returns (uint256 requiredBond_) {

747:     function claimCredit(address _recipient) external {

767:     function getChallengerDuration(uint256 _claimIndex) public view returns (Duration duration_) {

789:     function claimDataLen() external view returns (uint256 len_) {

798:     function absolutePrestate() external view returns (Claim absolutePrestate_) {

803:     function maxGameDepth() external view returns (uint256 maxGameDepth_) {

808:     function splitDepth() external view returns (uint256 splitDepth_) {

813:     function maxClockDuration() external view returns (Duration maxClockDuration_) {

818:     function clockExtension() external view returns (Duration clockExtension_) {

823:     function vm() external view returns (IBigStepper vm_) {

828:     function weth() external view returns (IDelayedWETH weth_) {

833:     function anchorStateRegistry() external view returns (IAnchorStateRegistry registry_) {

838:     function l2ChainId() external view returns (uint256 l2ChainId_) {

849:     function _distributeBond(address _recipient, ClaimData storage _bonded) internal {

931:     function _findStartingAndDisputedOutputs(uint256 _start)

995:     function _findLocalContext(uint256 _claimIndex) internal view returns (Hash uuid_) {

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="NC-10"></a>[NC-10] Change int to int256
Throughout the code base, some variables are declared as `int`. To favor explicitness, consider changing all instances of `int` to `int256`

*Instances (3)*:
```solidity
File: src/cannon/PreimageOracle.sol

289:                     0x0A, // point evaluation precompile address

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

466:         ResolutionCheckpoint storage checkpoint = resolutionCheckpoints[_claimIndex];

593:         ResolutionCheckpoint memory checkpoint = resolutionCheckpoints[_claimIndex];

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="NC-11"></a>[NC-11] Lack of checks in setters
Be it sanity checks (like checks against `0`-values) or initial setting checks: it's best for Setter functions to have them

*Instances (2)*:
```solidity
File: src/dispute/DisputeGameFactory.sol

206:         initBonds[_gameType] = _initBond;
             emit InitBondUpdated(_gameType, _initBond);
         }
     }
     

210: 

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol)

### <a name="NC-12"></a>[NC-12] Incomplete NatSpec: `@param` is missing on actually documented functions
The following functions are missing `@param` NatSpec comments.

*Instances (5)*:
```solidity
File: src/cannon/PreimageOracle.sol

416:     /// @notice Initialize a large preimage proposal. Must be called before adding any leaves.
         function initLPP(uint256 _uuid, uint32 _partOffset, uint32 _claimedSize) external payable {

439:     /// @notice Adds a contiguous list of keccak state matrices to the merkle tree.
         function addLeavesLPP(
             uint256 _uuid,
             uint256 _inputStartBlock,
             bytes calldata _input,
             bytes32[] calldata _stateCommitments,
             bool _finalize

567:     /// @notice Challenge a keccak256 block that was committed to in the merkle tree.
         function challengeLPP(
             address _claimant,
             uint256 _uuid,
             LibKeccak.StateMatrix memory _stateMatrix,
             Leaf calldata _preState,
             bytes32[] calldata _preStateProof,
             Leaf calldata _postState,
             bytes32[] calldata _postStateProof

608:     /// @notice Challenge the first keccak256 block that was absorbed.
         function challengeFirstLPP(
             address _claimant,
             uint256 _uuid,
             Leaf calldata _postState,
             bytes32[] calldata _postStateProof

639:     /// @notice Finalize a large preimage proposal after the challenge period has passed.
         function squeezeLPP(
             address _claimant,
             uint256 _uuid,
             LibKeccak.StateMatrix memory _stateMatrix,
             Leaf calldata _preState,
             bytes32[] calldata _preStateProof,
             Leaf calldata _postState,
             bytes32[] calldata _postStateProof

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

### <a name="NC-13"></a>[NC-13] Incomplete NatSpec: `@return` is missing on actually documented functions
The following functions are missing `@return` NatSpec comments.

*Instances (1)*:
```solidity
File: src/cannon/MIPS.sol

634:     /// @notice Executes a single step of the vm.
         ///         Will revert if any required input state is missing.
         /// @param _stateData The encoded state witness data.
         /// @param _proof The encoded proof data for leaves within the MIPS VM's memory.
         /// @param _localContext The local key context for the preimage oracle. Optional, can be set as a constant
         ///                      if the caller only requires one set of local keys.
         function step(bytes calldata _stateData, bytes calldata _proof, bytes32 _localContext) public returns (bytes32) {

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/MIPS.sol)

### <a name="NC-14"></a>[NC-14] Use a `modifier` instead of a `require/if` statement for a special `msg.sender` actor
If a function is supposed to be access-controlled, a `modifier` should be used instead of a `require/if` statement for more readability.

*Instances (2)*:
```solidity
File: src/cannon/PreimageOracle.sol

422:         if (msg.sender != tx.origin) revert NotEOA();

465:         if (msg.sender != tx.origin) revert NotEOA();

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

### <a name="NC-15"></a>[NC-15] Constant state variables defined more than once
Rather than redefining state variable constant, consider using a library to store all constants as this will prevent data redundancy

*Instances (4)*:
```solidity
File: src/cannon/MIPS.sol

47:     string public constant version = "1.0.1";

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/MIPS.sol)

```solidity
File: src/cannon/PreimageOracle.sol

33:     string public constant version = "1.0.0";

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

```solidity
File: src/dispute/DisputeGameFactory.sol

25:     string public constant version = "1.0.0";

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

73:     string public constant version = "1.2.0";

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="NC-16"></a>[NC-16] Consider using named mappings
Consider moving to solidity version 0.8.18 or later, and using [named mappings](https://ethereum.stackexchange.com/questions/51629/how-to-name-the-arguments-in-mapping/145555#145555) to make it easier to understand the purpose of each mapping

*Instances (16)*:
```solidity
File: src/cannon/PreimageOracle.sol

40:     mapping(bytes32 => uint256) public preimageLengths;

42:     mapping(bytes32 => mapping(uint256 => bytes32)) public preimageParts;

44:     mapping(bytes32 => mapping(uint256 => bool)) public preimagePartOk;

74:     mapping(address => mapping(uint256 => bytes32[KECCAK_TREE_DEPTH])) public proposalBranches;

77:     mapping(address => mapping(uint256 => LPPMetaData)) public proposalMetadata;

79:     mapping(address => mapping(uint256 => uint256)) public proposalBonds;

81:     mapping(address => mapping(uint256 => bytes32)) public proposalParts;

83:     mapping(address => mapping(uint256 => uint64[])) public proposalBlocks;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

```solidity
File: src/dispute/DisputeGameFactory.sol

28:     mapping(GameType => IDisputeGame) public gameImpls;

31:     mapping(GameType => uint256) public initBonds;

35:     mapping(Hash => GameId) internal _disputeGames;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

98:     mapping(address => uint256) public credit;

101:     mapping(Hash => bool) public claims;

104:     mapping(uint256 => uint256[]) public subgames;

107:     mapping(uint256 => bool) public resolvedSubgames;

110:     mapping(uint256 => ResolutionCheckpoint) public resolutionCheckpoints;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="NC-17"></a>[NC-17] Adding a `return` statement when the function defines a named return variable, is redundant

*Instances (51)*:
```solidity
File: src/cannon/MIPS.sol

74:     /// @notice Extends the value leftwards with its most significant bit (sign extension).
        function SE(uint32 _dat, uint32 _idx) internal pure returns (uint32 out_) {
            unchecked {
                bool isSigned = (_dat >> (_idx - 1)) != 0;
                uint256 signed = ((1 << (32 - _idx)) - 1) << _idx;
                uint256 mask = (1 << _idx) - 1;
                return uint32(_dat & mask | (isSigned ? signed : 0));

148:     /// @notice Handles a syscall.
         /// @param _localContext The local key context for the preimage oracle.
         /// @return out_ The hashed MIPS state.
         function handleSyscall(bytes32 _localContext) internal returns (bytes32 out_) {
             unchecked {
                 // Load state from memory
                 State memory state;
                 assembly {
                     state := 0x80
                 }
     
                 // Load the syscall number from the registers
                 uint32 syscall_no = state.registers[2];
                 uint32 v0 = 0;
                 uint32 v1 = 0;
     
                 // Load the syscall arguments from the registers
                 uint32 a0 = state.registers[4];
                 uint32 a1 = state.registers[5];
                 uint32 a2 = state.registers[6];
     
                 // mmap: Allocates a page from the heap.
                 if (syscall_no == 4090) {
                     uint32 sz = a1;
                     if (sz & 4095 != 0) {
                         // adjust size to align with page size
                         sz += 4096 - (sz & 4095);
                     }
                     if (a0 == 0) {
                         v0 = state.heap;
                         state.heap += sz;
                     } else {
                         v0 = a0;
                     }
                 }
                 // brk: Returns a fixed address for the program break at 0x40000000
                 else if (syscall_no == 4045) {
                     v0 = BRK_START;
                 }
                 // clone (not supported) returns 1
                 else if (syscall_no == 4120) {
                     v0 = 1;
                 }
                 // exit group: Sets the Exited and ExitCode states to true and argument 0.
                 else if (syscall_no == 4246) {
                     state.exited = true;
                     state.exitCode = uint8(a0);
                     return outputState();

517:     /// @notice Computes the offset of the proof in the calldata.
         /// @param _proofIndex The index of the proof in the calldata.
         /// @return offset_ The offset of the proof in the calldata.
         function proofOffset(uint8 _proofIndex) internal pure returns (uint256 offset_) {
             unchecked {
                 // A proof of 32 bit memory, with 32-byte leaf values, is (32-5)=27 bytes32 entries.
                 // And the leaf value itself needs to be encoded as well. And proof.offset == 420
                 offset_ = 420 + (uint256(_proofIndex) * (28 * 32));
                 uint256 s = 0;
                 assembly {
                     s := calldatasize()
                 }
                 require(s >= (offset_ + 28 * 32), "check that there is enough calldata");
                 return offset_;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;
                     }
                     // divu
                     else if (func == 0x1b) {
                         return rs;
                     }
                     // The rest includes transformed R-type arith imm instructions
                     // add
                     else if (func == 0x20) {
                         return (rs + rt);
                     }
                     // addu
                     else if (func == 0x21) {
                         return (rs + rt);
                     }
                     // sub
                     else if (func == 0x22) {
                         return (rs - rt);
                     }
                     // subu
                     else if (func == 0x23) {
                         return (rs - rt);
                     }
                     // and
                     else if (func == 0x24) {
                         return (rs & rt);
                     }
                     // or
                     else if (func == 0x25) {
                         return (rs | rt);
                     }
                     // xor
                     else if (func == 0x26) {
                         return (rs ^ rt);
                     }
                     // nor
                     else if (func == 0x27) {
                         return ~(rs | rt);
                     }
                     // slti
                     else if (func == 0x2a) {
                         return int32(rs) < int32(rt) ? 1 : 0;
                     }
                     // sltiu
                     else if (func == 0x2b) {
                         return rs < rt ? 1 : 0;
                     } else {
                         revert("invalid instruction");
                     }
                 } else {
                     // SPECIAL2
                     if (opcode == 0x1C) {
                         uint32 func = insn & 0x3f; // 6-bits
                         // mul
                         if (func == 0x2) {
                             return uint32(int32(rs) * int32(rt));
                         }
                         // clz, clo
                         else if (func == 0x20 || func == 0x21) {
                             if (func == 0x20) {
                                 rs = ~rs;
                             }
                             uint32 i = 0;
                             while (rs & 0x80000000 != 0) {
                                 i++;
                                 rs <<= 1;
                             }
                             return i;
                         }
                     }
                     // lui
                     else if (opcode == 0x0F) {
                         return rt << 16;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;
                     }
                     // divu
                     else if (func == 0x1b) {
                         return rs;
                     }
                     // The rest includes transformed R-type arith imm instructions
                     // add
                     else if (func == 0x20) {
                         return (rs + rt);
                     }
                     // addu
                     else if (func == 0x21) {
                         return (rs + rt);
                     }
                     // sub
                     else if (func == 0x22) {
                         return (rs - rt);
                     }
                     // subu
                     else if (func == 0x23) {
                         return (rs - rt);
                     }
                     // and
                     else if (func == 0x24) {
                         return (rs & rt);
                     }
                     // or
                     else if (func == 0x25) {
                         return (rs | rt);
                     }
                     // xor
                     else if (func == 0x26) {
                         return (rs ^ rt);
                     }
                     // nor
                     else if (func == 0x27) {
                         return ~(rs | rt);
                     }
                     // slti
                     else if (func == 0x2a) {
                         return int32(rs) < int32(rt) ? 1 : 0;
                     }
                     // sltiu
                     else if (func == 0x2b) {
                         return rs < rt ? 1 : 0;
                     } else {
                         revert("invalid instruction");
                     }
                 } else {
                     // SPECIAL2
                     if (opcode == 0x1C) {
                         uint32 func = insn & 0x3f; // 6-bits
                         // mul
                         if (func == 0x2) {
                             return uint32(int32(rs) * int32(rt));
                         }
                         // clz, clo
                         else if (func == 0x20 || func == 0x21) {
                             if (func == 0x20) {
                                 rs = ~rs;
                             }
                             uint32 i = 0;
                             while (rs & 0x80000000 != 0) {
                                 i++;
                                 rs <<= 1;
                             }
                             return i;
                         }
                     }
                     // lui
                     else if (opcode == 0x0F) {
                         return rt << 16;
                     }
                     // lb
                     else if (opcode == 0x20) {
                         return SE((mem >> (24 - (rs & 3) * 8)) & 0xFF, 8);

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;
                     }
                     // divu
                     else if (func == 0x1b) {
                         return rs;
                     }
                     // The rest includes transformed R-type arith imm instructions
                     // add
                     else if (func == 0x20) {
                         return (rs + rt);
                     }
                     // addu
                     else if (func == 0x21) {
                         return (rs + rt);
                     }
                     // sub
                     else if (func == 0x22) {
                         return (rs - rt);
                     }
                     // subu
                     else if (func == 0x23) {
                         return (rs - rt);
                     }
                     // and
                     else if (func == 0x24) {
                         return (rs & rt);
                     }
                     // or
                     else if (func == 0x25) {
                         return (rs | rt);
                     }
                     // xor
                     else if (func == 0x26) {
                         return (rs ^ rt);
                     }
                     // nor
                     else if (func == 0x27) {
                         return ~(rs | rt);
                     }
                     // slti
                     else if (func == 0x2a) {
                         return int32(rs) < int32(rt) ? 1 : 0;
                     }
                     // sltiu
                     else if (func == 0x2b) {
                         return rs < rt ? 1 : 0;
                     } else {
                         revert("invalid instruction");
                     }
                 } else {
                     // SPECIAL2
                     if (opcode == 0x1C) {
                         uint32 func = insn & 0x3f; // 6-bits
                         // mul
                         if (func == 0x2) {
                             return uint32(int32(rs) * int32(rt));

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;
                     }
                     // divu
                     else if (func == 0x1b) {
                         return rs;
                     }
                     // The rest includes transformed R-type arith imm instructions
                     // add
                     else if (func == 0x20) {
                         return (rs + rt);
                     }
                     // addu
                     else if (func == 0x21) {
                         return (rs + rt);
                     }
                     // sub
                     else if (func == 0x22) {
                         return (rs - rt);
                     }
                     // subu
                     else if (func == 0x23) {
                         return (rs - rt);
                     }
                     // and
                     else if (func == 0x24) {
                         return (rs & rt);
                     }
                     // or
                     else if (func == 0x25) {
                         return (rs | rt);
                     }
                     // xor
                     else if (func == 0x26) {
                         return (rs ^ rt);
                     }
                     // nor
                     else if (func == 0x27) {
                         return ~(rs | rt);
                     }
                     // slti
                     else if (func == 0x2a) {
                         return int32(rs) < int32(rt) ? 1 : 0;
                     }
                     // sltiu
                     else if (func == 0x2b) {
                         return rs < rt ? 1 : 0;
                     } else {
                         revert("invalid instruction");
                     }
                 } else {
                     // SPECIAL2
                     if (opcode == 0x1C) {
                         uint32 func = insn & 0x3f; // 6-bits
                         // mul
                         if (func == 0x2) {
                             return uint32(int32(rs) * int32(rt));
                         }
                         // clz, clo
                         else if (func == 0x20 || func == 0x21) {
                             if (func == 0x20) {
                                 rs = ~rs;
                             }
                             uint32 i = 0;
                             while (rs & 0x80000000 != 0) {
                                 i++;
                                 rs <<= 1;
                             }
                             return i;
                         }
                     }
                     // lui
                     else if (opcode == 0x0F) {
                         return rt << 16;
                     }
                     // lb
                     else if (opcode == 0x20) {
                         return SE((mem >> (24 - (rs & 3) * 8)) & 0xFF, 8);
                     }
                     // lh
                     else if (opcode == 0x21) {
                         return SE((mem >> (16 - (rs & 2) * 8)) & 0xFFFF, 16);

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;
                     }
                     // divu
                     else if (func == 0x1b) {
                         return rs;
                     }
                     // The rest includes transformed R-type arith imm instructions
                     // add
                     else if (func == 0x20) {
                         return (rs + rt);
                     }
                     // addu
                     else if (func == 0x21) {
                         return (rs + rt);
                     }
                     // sub
                     else if (func == 0x22) {
                         return (rs - rt);
                     }
                     // subu
                     else if (func == 0x23) {
                         return (rs - rt);
                     }
                     // and
                     else if (func == 0x24) {
                         return (rs & rt);
                     }
                     // or
                     else if (func == 0x25) {
                         return (rs | rt);
                     }
                     // xor
                     else if (func == 0x26) {
                         return (rs ^ rt);
                     }
                     // nor
                     else if (func == 0x27) {
                         return ~(rs | rt);
                     }
                     // slti
                     else if (func == 0x2a) {
                         return int32(rs) < int32(rt) ? 1 : 0;
                     }
                     // sltiu
                     else if (func == 0x2b) {
                         return rs < rt ? 1 : 0;
                     } else {
                         revert("invalid instruction");
                     }
                 } else {
                     // SPECIAL2
                     if (opcode == 0x1C) {
                         uint32 func = insn & 0x3f; // 6-bits
                         // mul
                         if (func == 0x2) {
                             return uint32(int32(rs) * int32(rt));
                         }
                         // clz, clo
                         else if (func == 0x20 || func == 0x21) {
                             if (func == 0x20) {
                                 rs = ~rs;
                             }
                             uint32 i = 0;
                             while (rs & 0x80000000 != 0) {
                                 i++;
                                 rs <<= 1;
                             }
                             return i;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;
                     }
                     // divu
                     else if (func == 0x1b) {
                         return rs;
                     }
                     // The rest includes transformed R-type arith imm instructions
                     // add
                     else if (func == 0x20) {
                         return (rs + rt);
                     }
                     // addu
                     else if (func == 0x21) {
                         return (rs + rt);
                     }
                     // sub
                     else if (func == 0x22) {
                         return (rs - rt);
                     }
                     // subu
                     else if (func == 0x23) {
                         return (rs - rt);
                     }
                     // and
                     else if (func == 0x24) {
                         return (rs & rt);
                     }
                     // or
                     else if (func == 0x25) {
                         return (rs | rt);
                     }
                     // xor
                     else if (func == 0x26) {
                         return (rs ^ rt);
                     }
                     // nor
                     else if (func == 0x27) {
                         return ~(rs | rt);
                     }
                     // slti
                     else if (func == 0x2a) {
                         return int32(rs) < int32(rt) ? 1 : 0;
                     }
                     // sltiu
                     else if (func == 0x2b) {
                         return rs < rt ? 1 : 0;
                     } else {
                         revert("invalid instruction");
                     }
                 } else {
                     // SPECIAL2
                     if (opcode == 0x1C) {
                         uint32 func = insn & 0x3f; // 6-bits
                         // mul
                         if (func == 0x2) {
                             return uint32(int32(rs) * int32(rt));
                         }
                         // clz, clo
                         else if (func == 0x20 || func == 0x21) {
                             if (func == 0x20) {
                                 rs = ~rs;
                             }
                             uint32 i = 0;
                             while (rs & 0x80000000 != 0) {
                                 i++;
                                 rs <<= 1;
                             }
                             return i;
                         }
                     }
                     // lui
                     else if (opcode == 0x0F) {
                         return rt << 16;
                     }
                     // lb
                     else if (opcode == 0x20) {
                         return SE((mem >> (24 - (rs & 3) * 8)) & 0xFF, 8);
                     }
                     // lh
                     else if (opcode == 0x21) {
                         return SE((mem >> (16 - (rs & 2) * 8)) & 0xFFFF, 16);
                     }
                     // lwl
                     else if (opcode == 0x22) {
                         uint32 val = mem << ((rs & 3) * 8);
                         uint32 mask = uint32(0xFFFFFFFF) << ((rs & 3) * 8);
                         return (rt & ~mask) | val;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;
                     }
                     // divu
                     else if (func == 0x1b) {
                         return rs;
                     }
                     // The rest includes transformed R-type arith imm instructions
                     // add
                     else if (func == 0x20) {
                         return (rs + rt);
                     }
                     // addu
                     else if (func == 0x21) {
                         return (rs + rt);
                     }
                     // sub
                     else if (func == 0x22) {
                         return (rs - rt);
                     }
                     // subu
                     else if (func == 0x23) {
                         return (rs - rt);
                     }
                     // and
                     else if (func == 0x24) {
                         return (rs & rt);
                     }
                     // or
                     else if (func == 0x25) {
                         return (rs | rt);
                     }
                     // xor
                     else if (func == 0x26) {
                         return (rs ^ rt);
                     }
                     // nor
                     else if (func == 0x27) {
                         return ~(rs | rt);
                     }
                     // slti
                     else if (func == 0x2a) {
                         return int32(rs) < int32(rt) ? 1 : 0;
                     }
                     // sltiu
                     else if (func == 0x2b) {
                         return rs < rt ? 1 : 0;
                     } else {
                         revert("invalid instruction");
                     }
                 } else {
                     // SPECIAL2
                     if (opcode == 0x1C) {
                         uint32 func = insn & 0x3f; // 6-bits
                         // mul
                         if (func == 0x2) {
                             return uint32(int32(rs) * int32(rt));
                         }
                         // clz, clo
                         else if (func == 0x20 || func == 0x21) {
                             if (func == 0x20) {
                                 rs = ~rs;
                             }
                             uint32 i = 0;
                             while (rs & 0x80000000 != 0) {
                                 i++;
                                 rs <<= 1;
                             }
                             return i;
                         }
                     }
                     // lui
                     else if (opcode == 0x0F) {
                         return rt << 16;
                     }
                     // lb
                     else if (opcode == 0x20) {
                         return SE((mem >> (24 - (rs & 3) * 8)) & 0xFF, 8);
                     }
                     // lh
                     else if (opcode == 0x21) {
                         return SE((mem >> (16 - (rs & 2) * 8)) & 0xFFFF, 16);
                     }
                     // lwl
                     else if (opcode == 0x22) {
                         uint32 val = mem << ((rs & 3) * 8);
                         uint32 mask = uint32(0xFFFFFFFF) << ((rs & 3) * 8);
                         return (rt & ~mask) | val;
                     }
                     // lw
                     else if (opcode == 0x23) {
                         return mem;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;
                     }
                     // divu
                     else if (func == 0x1b) {
                         return rs;
                     }
                     // The rest includes transformed R-type arith imm instructions
                     // add
                     else if (func == 0x20) {
                         return (rs + rt);
                     }
                     // addu
                     else if (func == 0x21) {
                         return (rs + rt);
                     }
                     // sub
                     else if (func == 0x22) {
                         return (rs - rt);
                     }
                     // subu
                     else if (func == 0x23) {
                         return (rs - rt);
                     }
                     // and
                     else if (func == 0x24) {
                         return (rs & rt);
                     }
                     // or
                     else if (func == 0x25) {
                         return (rs | rt);
                     }
                     // xor
                     else if (func == 0x26) {
                         return (rs ^ rt);
                     }
                     // nor
                     else if (func == 0x27) {
                         return ~(rs | rt);
                     }
                     // slti
                     else if (func == 0x2a) {
                         return int32(rs) < int32(rt) ? 1 : 0;
                     }
                     // sltiu
                     else if (func == 0x2b) {
                         return rs < rt ? 1 : 0;
                     } else {
                         revert("invalid instruction");
                     }
                 } else {
                     // SPECIAL2
                     if (opcode == 0x1C) {
                         uint32 func = insn & 0x3f; // 6-bits
                         // mul
                         if (func == 0x2) {
                             return uint32(int32(rs) * int32(rt));
                         }
                         // clz, clo
                         else if (func == 0x20 || func == 0x21) {
                             if (func == 0x20) {
                                 rs = ~rs;
                             }
                             uint32 i = 0;
                             while (rs & 0x80000000 != 0) {
                                 i++;
                                 rs <<= 1;
                             }
                             return i;
                         }
                     }
                     // lui
                     else if (opcode == 0x0F) {
                         return rt << 16;
                     }
                     // lb
                     else if (opcode == 0x20) {
                         return SE((mem >> (24 - (rs & 3) * 8)) & 0xFF, 8);
                     }
                     // lh
                     else if (opcode == 0x21) {
                         return SE((mem >> (16 - (rs & 2) * 8)) & 0xFFFF, 16);
                     }
                     // lwl
                     else if (opcode == 0x22) {
                         uint32 val = mem << ((rs & 3) * 8);
                         uint32 mask = uint32(0xFFFFFFFF) << ((rs & 3) * 8);
                         return (rt & ~mask) | val;
                     }
                     // lw
                     else if (opcode == 0x23) {
                         return mem;
                     }
                     // lbu
                     else if (opcode == 0x24) {
                         return (mem >> (24 - (rs & 3) * 8)) & 0xFF;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;
                     }
                     // divu
                     else if (func == 0x1b) {
                         return rs;
                     }
                     // The rest includes transformed R-type arith imm instructions
                     // add
                     else if (func == 0x20) {
                         return (rs + rt);
                     }
                     // addu
                     else if (func == 0x21) {
                         return (rs + rt);
                     }
                     // sub
                     else if (func == 0x22) {
                         return (rs - rt);
                     }
                     // subu
                     else if (func == 0x23) {
                         return (rs - rt);
                     }
                     // and
                     else if (func == 0x24) {
                         return (rs & rt);
                     }
                     // or
                     else if (func == 0x25) {
                         return (rs | rt);
                     }
                     // xor
                     else if (func == 0x26) {
                         return (rs ^ rt);
                     }
                     // nor
                     else if (func == 0x27) {
                         return ~(rs | rt);
                     }
                     // slti
                     else if (func == 0x2a) {
                         return int32(rs) < int32(rt) ? 1 : 0;
                     }
                     // sltiu
                     else if (func == 0x2b) {
                         return rs < rt ? 1 : 0;
                     } else {
                         revert("invalid instruction");
                     }
                 } else {
                     // SPECIAL2
                     if (opcode == 0x1C) {
                         uint32 func = insn & 0x3f; // 6-bits
                         // mul
                         if (func == 0x2) {
                             return uint32(int32(rs) * int32(rt));
                         }
                         // clz, clo
                         else if (func == 0x20 || func == 0x21) {
                             if (func == 0x20) {
                                 rs = ~rs;
                             }
                             uint32 i = 0;
                             while (rs & 0x80000000 != 0) {
                                 i++;
                                 rs <<= 1;
                             }
                             return i;
                         }
                     }
                     // lui
                     else if (opcode == 0x0F) {
                         return rt << 16;
                     }
                     // lb
                     else if (opcode == 0x20) {
                         return SE((mem >> (24 - (rs & 3) * 8)) & 0xFF, 8);
                     }
                     // lh
                     else if (opcode == 0x21) {
                         return SE((mem >> (16 - (rs & 2) * 8)) & 0xFFFF, 16);
                     }
                     // lwl
                     else if (opcode == 0x22) {
                         uint32 val = mem << ((rs & 3) * 8);
                         uint32 mask = uint32(0xFFFFFFFF) << ((rs & 3) * 8);
                         return (rt & ~mask) | val;
                     }
                     // lw
                     else if (opcode == 0x23) {
                         return mem;
                     }
                     // lbu
                     else if (opcode == 0x24) {
                         return (mem >> (24 - (rs & 3) * 8)) & 0xFF;
                     }
                     //  lhu
                     else if (opcode == 0x25) {
                         return (mem >> (16 - (rs & 2) * 8)) & 0xFFFF;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;
                     }
                     // divu
                     else if (func == 0x1b) {
                         return rs;
                     }
                     // The rest includes transformed R-type arith imm instructions
                     // add
                     else if (func == 0x20) {
                         return (rs + rt);
                     }
                     // addu
                     else if (func == 0x21) {
                         return (rs + rt);
                     }
                     // sub
                     else if (func == 0x22) {
                         return (rs - rt);
                     }
                     // subu
                     else if (func == 0x23) {
                         return (rs - rt);
                     }
                     // and
                     else if (func == 0x24) {
                         return (rs & rt);
                     }
                     // or
                     else if (func == 0x25) {
                         return (rs | rt);
                     }
                     // xor
                     else if (func == 0x26) {
                         return (rs ^ rt);
                     }
                     // nor
                     else if (func == 0x27) {
                         return ~(rs | rt);
                     }
                     // slti
                     else if (func == 0x2a) {
                         return int32(rs) < int32(rt) ? 1 : 0;
                     }
                     // sltiu
                     else if (func == 0x2b) {
                         return rs < rt ? 1 : 0;
                     } else {
                         revert("invalid instruction");
                     }
                 } else {
                     // SPECIAL2
                     if (opcode == 0x1C) {
                         uint32 func = insn & 0x3f; // 6-bits
                         // mul
                         if (func == 0x2) {
                             return uint32(int32(rs) * int32(rt));
                         }
                         // clz, clo
                         else if (func == 0x20 || func == 0x21) {
                             if (func == 0x20) {
                                 rs = ~rs;
                             }
                             uint32 i = 0;
                             while (rs & 0x80000000 != 0) {
                                 i++;
                                 rs <<= 1;
                             }
                             return i;
                         }
                     }
                     // lui
                     else if (opcode == 0x0F) {
                         return rt << 16;
                     }
                     // lb
                     else if (opcode == 0x20) {
                         return SE((mem >> (24 - (rs & 3) * 8)) & 0xFF, 8);
                     }
                     // lh
                     else if (opcode == 0x21) {
                         return SE((mem >> (16 - (rs & 2) * 8)) & 0xFFFF, 16);
                     }
                     // lwl
                     else if (opcode == 0x22) {
                         uint32 val = mem << ((rs & 3) * 8);
                         uint32 mask = uint32(0xFFFFFFFF) << ((rs & 3) * 8);
                         return (rt & ~mask) | val;
                     }
                     // lw
                     else if (opcode == 0x23) {
                         return mem;
                     }
                     // lbu
                     else if (opcode == 0x24) {
                         return (mem >> (24 - (rs & 3) * 8)) & 0xFF;
                     }
                     //  lhu
                     else if (opcode == 0x25) {
                         return (mem >> (16 - (rs & 2) * 8)) & 0xFFFF;
                     }
                     //  lwr
                     else if (opcode == 0x26) {
                         uint32 val = mem >> (24 - (rs & 3) * 8);
                         uint32 mask = uint32(0xFFFFFFFF) >> (24 - (rs & 3) * 8);
                         return (rt & ~mask) | val;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;
                     }
                     // divu
                     else if (func == 0x1b) {
                         return rs;
                     }
                     // The rest includes transformed R-type arith imm instructions
                     // add
                     else if (func == 0x20) {
                         return (rs + rt);
                     }
                     // addu
                     else if (func == 0x21) {
                         return (rs + rt);
                     }
                     // sub
                     else if (func == 0x22) {
                         return (rs - rt);
                     }
                     // subu
                     else if (func == 0x23) {
                         return (rs - rt);
                     }
                     // and
                     else if (func == 0x24) {
                         return (rs & rt);
                     }
                     // or
                     else if (func == 0x25) {
                         return (rs | rt);
                     }
                     // xor
                     else if (func == 0x26) {
                         return (rs ^ rt);
                     }
                     // nor
                     else if (func == 0x27) {
                         return ~(rs | rt);
                     }
                     // slti
                     else if (func == 0x2a) {
                         return int32(rs) < int32(rt) ? 1 : 0;
                     }
                     // sltiu
                     else if (func == 0x2b) {
                         return rs < rt ? 1 : 0;
                     } else {
                         revert("invalid instruction");
                     }
                 } else {
                     // SPECIAL2
                     if (opcode == 0x1C) {
                         uint32 func = insn & 0x3f; // 6-bits
                         // mul
                         if (func == 0x2) {
                             return uint32(int32(rs) * int32(rt));
                         }
                         // clz, clo
                         else if (func == 0x20 || func == 0x21) {
                             if (func == 0x20) {
                                 rs = ~rs;
                             }
                             uint32 i = 0;
                             while (rs & 0x80000000 != 0) {
                                 i++;
                                 rs <<= 1;
                             }
                             return i;
                         }
                     }
                     // lui
                     else if (opcode == 0x0F) {
                         return rt << 16;
                     }
                     // lb
                     else if (opcode == 0x20) {
                         return SE((mem >> (24 - (rs & 3) * 8)) & 0xFF, 8);
                     }
                     // lh
                     else if (opcode == 0x21) {
                         return SE((mem >> (16 - (rs & 2) * 8)) & 0xFFFF, 16);
                     }
                     // lwl
                     else if (opcode == 0x22) {
                         uint32 val = mem << ((rs & 3) * 8);
                         uint32 mask = uint32(0xFFFFFFFF) << ((rs & 3) * 8);
                         return (rt & ~mask) | val;
                     }
                     // lw
                     else if (opcode == 0x23) {
                         return mem;
                     }
                     // lbu
                     else if (opcode == 0x24) {
                         return (mem >> (24 - (rs & 3) * 8)) & 0xFF;
                     }
                     //  lhu
                     else if (opcode == 0x25) {
                         return (mem >> (16 - (rs & 2) * 8)) & 0xFFFF;
                     }
                     //  lwr
                     else if (opcode == 0x26) {
                         uint32 val = mem >> (24 - (rs & 3) * 8);
                         uint32 mask = uint32(0xFFFFFFFF) >> (24 - (rs & 3) * 8);
                         return (rt & ~mask) | val;
                     }
                     //  sb
                     else if (opcode == 0x28) {
                         uint32 val = (rt & 0xFF) << (24 - (rs & 3) * 8);
                         uint32 mask = 0xFFFFFFFF ^ uint32(0xFF << (24 - (rs & 3) * 8));
                         return (mem & mask) | val;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;
                     }
                     // divu
                     else if (func == 0x1b) {
                         return rs;
                     }
                     // The rest includes transformed R-type arith imm instructions
                     // add
                     else if (func == 0x20) {
                         return (rs + rt);
                     }
                     // addu
                     else if (func == 0x21) {
                         return (rs + rt);
                     }
                     // sub
                     else if (func == 0x22) {
                         return (rs - rt);
                     }
                     // subu
                     else if (func == 0x23) {
                         return (rs - rt);
                     }
                     // and
                     else if (func == 0x24) {
                         return (rs & rt);
                     }
                     // or
                     else if (func == 0x25) {
                         return (rs | rt);
                     }
                     // xor
                     else if (func == 0x26) {
                         return (rs ^ rt);
                     }
                     // nor
                     else if (func == 0x27) {
                         return ~(rs | rt);
                     }
                     // slti
                     else if (func == 0x2a) {
                         return int32(rs) < int32(rt) ? 1 : 0;
                     }
                     // sltiu
                     else if (func == 0x2b) {
                         return rs < rt ? 1 : 0;
                     } else {
                         revert("invalid instruction");
                     }
                 } else {
                     // SPECIAL2
                     if (opcode == 0x1C) {
                         uint32 func = insn & 0x3f; // 6-bits
                         // mul
                         if (func == 0x2) {
                             return uint32(int32(rs) * int32(rt));
                         }
                         // clz, clo
                         else if (func == 0x20 || func == 0x21) {
                             if (func == 0x20) {
                                 rs = ~rs;
                             }
                             uint32 i = 0;
                             while (rs & 0x80000000 != 0) {
                                 i++;
                                 rs <<= 1;
                             }
                             return i;
                         }
                     }
                     // lui
                     else if (opcode == 0x0F) {
                         return rt << 16;
                     }
                     // lb
                     else if (opcode == 0x20) {
                         return SE((mem >> (24 - (rs & 3) * 8)) & 0xFF, 8);
                     }
                     // lh
                     else if (opcode == 0x21) {
                         return SE((mem >> (16 - (rs & 2) * 8)) & 0xFFFF, 16);
                     }
                     // lwl
                     else if (opcode == 0x22) {
                         uint32 val = mem << ((rs & 3) * 8);
                         uint32 mask = uint32(0xFFFFFFFF) << ((rs & 3) * 8);
                         return (rt & ~mask) | val;
                     }
                     // lw
                     else if (opcode == 0x23) {
                         return mem;
                     }
                     // lbu
                     else if (opcode == 0x24) {
                         return (mem >> (24 - (rs & 3) * 8)) & 0xFF;
                     }
                     //  lhu
                     else if (opcode == 0x25) {
                         return (mem >> (16 - (rs & 2) * 8)) & 0xFFFF;
                     }
                     //  lwr
                     else if (opcode == 0x26) {
                         uint32 val = mem >> (24 - (rs & 3) * 8);
                         uint32 mask = uint32(0xFFFFFFFF) >> (24 - (rs & 3) * 8);
                         return (rt & ~mask) | val;
                     }
                     //  sb
                     else if (opcode == 0x28) {
                         uint32 val = (rt & 0xFF) << (24 - (rs & 3) * 8);
                         uint32 mask = 0xFFFFFFFF ^ uint32(0xFF << (24 - (rs & 3) * 8));
                         return (mem & mask) | val;
                     }
                     //  sh
                     else if (opcode == 0x29) {
                         uint32 val = (rt & 0xFFFF) << (16 - (rs & 2) * 8);
                         uint32 mask = 0xFFFFFFFF ^ uint32(0xFFFF << (16 - (rs & 2) * 8));
                         return (mem & mask) | val;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;
                     }
                     // divu
                     else if (func == 0x1b) {
                         return rs;
                     }
                     // The rest includes transformed R-type arith imm instructions
                     // add
                     else if (func == 0x20) {
                         return (rs + rt);
                     }
                     // addu
                     else if (func == 0x21) {
                         return (rs + rt);
                     }
                     // sub
                     else if (func == 0x22) {
                         return (rs - rt);
                     }
                     // subu
                     else if (func == 0x23) {
                         return (rs - rt);
                     }
                     // and
                     else if (func == 0x24) {
                         return (rs & rt);
                     }
                     // or
                     else if (func == 0x25) {
                         return (rs | rt);
                     }
                     // xor
                     else if (func == 0x26) {
                         return (rs ^ rt);
                     }
                     // nor
                     else if (func == 0x27) {
                         return ~(rs | rt);
                     }
                     // slti
                     else if (func == 0x2a) {
                         return int32(rs) < int32(rt) ? 1 : 0;
                     }
                     // sltiu
                     else if (func == 0x2b) {
                         return rs < rt ? 1 : 0;
                     } else {
                         revert("invalid instruction");
                     }
                 } else {
                     // SPECIAL2
                     if (opcode == 0x1C) {
                         uint32 func = insn & 0x3f; // 6-bits
                         // mul
                         if (func == 0x2) {
                             return uint32(int32(rs) * int32(rt));
                         }
                         // clz, clo
                         else if (func == 0x20 || func == 0x21) {
                             if (func == 0x20) {
                                 rs = ~rs;
                             }
                             uint32 i = 0;
                             while (rs & 0x80000000 != 0) {
                                 i++;
                                 rs <<= 1;
                             }
                             return i;
                         }
                     }
                     // lui
                     else if (opcode == 0x0F) {
                         return rt << 16;
                     }
                     // lb
                     else if (opcode == 0x20) {
                         return SE((mem >> (24 - (rs & 3) * 8)) & 0xFF, 8);
                     }
                     // lh
                     else if (opcode == 0x21) {
                         return SE((mem >> (16 - (rs & 2) * 8)) & 0xFFFF, 16);
                     }
                     // lwl
                     else if (opcode == 0x22) {
                         uint32 val = mem << ((rs & 3) * 8);
                         uint32 mask = uint32(0xFFFFFFFF) << ((rs & 3) * 8);
                         return (rt & ~mask) | val;
                     }
                     // lw
                     else if (opcode == 0x23) {
                         return mem;
                     }
                     // lbu
                     else if (opcode == 0x24) {
                         return (mem >> (24 - (rs & 3) * 8)) & 0xFF;
                     }
                     //  lhu
                     else if (opcode == 0x25) {
                         return (mem >> (16 - (rs & 2) * 8)) & 0xFFFF;
                     }
                     //  lwr
                     else if (opcode == 0x26) {
                         uint32 val = mem >> (24 - (rs & 3) * 8);
                         uint32 mask = uint32(0xFFFFFFFF) >> (24 - (rs & 3) * 8);
                         return (rt & ~mask) | val;
                     }
                     //  sb
                     else if (opcode == 0x28) {
                         uint32 val = (rt & 0xFF) << (24 - (rs & 3) * 8);
                         uint32 mask = 0xFFFFFFFF ^ uint32(0xFF << (24 - (rs & 3) * 8));
                         return (mem & mask) | val;
                     }
                     //  sh
                     else if (opcode == 0x29) {
                         uint32 val = (rt & 0xFFFF) << (16 - (rs & 2) * 8);
                         uint32 mask = 0xFFFFFFFF ^ uint32(0xFFFF << (16 - (rs & 2) * 8));
                         return (mem & mask) | val;
                     }
                     //  swl
                     else if (opcode == 0x2a) {
                         uint32 val = rt >> ((rs & 3) * 8);
                         uint32 mask = uint32(0xFFFFFFFF) >> ((rs & 3) * 8);
                         return (mem & ~mask) | val;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;
                     }
                     // divu
                     else if (func == 0x1b) {
                         return rs;
                     }
                     // The rest includes transformed R-type arith imm instructions
                     // add
                     else if (func == 0x20) {
                         return (rs + rt);
                     }
                     // addu
                     else if (func == 0x21) {
                         return (rs + rt);
                     }
                     // sub
                     else if (func == 0x22) {
                         return (rs - rt);
                     }
                     // subu
                     else if (func == 0x23) {
                         return (rs - rt);
                     }
                     // and
                     else if (func == 0x24) {
                         return (rs & rt);
                     }
                     // or
                     else if (func == 0x25) {
                         return (rs | rt);
                     }
                     // xor
                     else if (func == 0x26) {
                         return (rs ^ rt);
                     }
                     // nor
                     else if (func == 0x27) {
                         return ~(rs | rt);
                     }
                     // slti
                     else if (func == 0x2a) {
                         return int32(rs) < int32(rt) ? 1 : 0;
                     }
                     // sltiu
                     else if (func == 0x2b) {
                         return rs < rt ? 1 : 0;
                     } else {
                         revert("invalid instruction");
                     }
                 } else {
                     // SPECIAL2
                     if (opcode == 0x1C) {
                         uint32 func = insn & 0x3f; // 6-bits
                         // mul
                         if (func == 0x2) {
                             return uint32(int32(rs) * int32(rt));
                         }
                         // clz, clo
                         else if (func == 0x20 || func == 0x21) {
                             if (func == 0x20) {
                                 rs = ~rs;
                             }
                             uint32 i = 0;
                             while (rs & 0x80000000 != 0) {
                                 i++;
                                 rs <<= 1;
                             }
                             return i;
                         }
                     }
                     // lui
                     else if (opcode == 0x0F) {
                         return rt << 16;
                     }
                     // lb
                     else if (opcode == 0x20) {
                         return SE((mem >> (24 - (rs & 3) * 8)) & 0xFF, 8);
                     }
                     // lh
                     else if (opcode == 0x21) {
                         return SE((mem >> (16 - (rs & 2) * 8)) & 0xFFFF, 16);
                     }
                     // lwl
                     else if (opcode == 0x22) {
                         uint32 val = mem << ((rs & 3) * 8);
                         uint32 mask = uint32(0xFFFFFFFF) << ((rs & 3) * 8);
                         return (rt & ~mask) | val;
                     }
                     // lw
                     else if (opcode == 0x23) {
                         return mem;
                     }
                     // lbu
                     else if (opcode == 0x24) {
                         return (mem >> (24 - (rs & 3) * 8)) & 0xFF;
                     }
                     //  lhu
                     else if (opcode == 0x25) {
                         return (mem >> (16 - (rs & 2) * 8)) & 0xFFFF;
                     }
                     //  lwr
                     else if (opcode == 0x26) {
                         uint32 val = mem >> (24 - (rs & 3) * 8);
                         uint32 mask = uint32(0xFFFFFFFF) >> (24 - (rs & 3) * 8);
                         return (rt & ~mask) | val;
                     }
                     //  sb
                     else if (opcode == 0x28) {
                         uint32 val = (rt & 0xFF) << (24 - (rs & 3) * 8);
                         uint32 mask = 0xFFFFFFFF ^ uint32(0xFF << (24 - (rs & 3) * 8));
                         return (mem & mask) | val;
                     }
                     //  sh
                     else if (opcode == 0x29) {
                         uint32 val = (rt & 0xFFFF) << (16 - (rs & 2) * 8);
                         uint32 mask = 0xFFFFFFFF ^ uint32(0xFFFF << (16 - (rs & 2) * 8));
                         return (mem & mask) | val;
                     }
                     //  swl
                     else if (opcode == 0x2a) {
                         uint32 val = rt >> ((rs & 3) * 8);
                         uint32 mask = uint32(0xFFFFFFFF) >> ((rs & 3) * 8);
                         return (mem & ~mask) | val;
                     }
                     //  sw
                     else if (opcode == 0x2b) {
                         return rt;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;
                     }
                     // divu
                     else if (func == 0x1b) {
                         return rs;
                     }
                     // The rest includes transformed R-type arith imm instructions
                     // add
                     else if (func == 0x20) {
                         return (rs + rt);
                     }
                     // addu
                     else if (func == 0x21) {
                         return (rs + rt);
                     }
                     // sub
                     else if (func == 0x22) {
                         return (rs - rt);
                     }
                     // subu
                     else if (func == 0x23) {
                         return (rs - rt);
                     }
                     // and
                     else if (func == 0x24) {
                         return (rs & rt);
                     }
                     // or
                     else if (func == 0x25) {
                         return (rs | rt);
                     }
                     // xor
                     else if (func == 0x26) {
                         return (rs ^ rt);
                     }
                     // nor
                     else if (func == 0x27) {
                         return ~(rs | rt);
                     }
                     // slti
                     else if (func == 0x2a) {
                         return int32(rs) < int32(rt) ? 1 : 0;
                     }
                     // sltiu
                     else if (func == 0x2b) {
                         return rs < rt ? 1 : 0;
                     } else {
                         revert("invalid instruction");
                     }
                 } else {
                     // SPECIAL2
                     if (opcode == 0x1C) {
                         uint32 func = insn & 0x3f; // 6-bits
                         // mul
                         if (func == 0x2) {
                             return uint32(int32(rs) * int32(rt));
                         }
                         // clz, clo
                         else if (func == 0x20 || func == 0x21) {
                             if (func == 0x20) {
                                 rs = ~rs;
                             }
                             uint32 i = 0;
                             while (rs & 0x80000000 != 0) {
                                 i++;
                                 rs <<= 1;
                             }
                             return i;
                         }
                     }
                     // lui
                     else if (opcode == 0x0F) {
                         return rt << 16;
                     }
                     // lb
                     else if (opcode == 0x20) {
                         return SE((mem >> (24 - (rs & 3) * 8)) & 0xFF, 8);
                     }
                     // lh
                     else if (opcode == 0x21) {
                         return SE((mem >> (16 - (rs & 2) * 8)) & 0xFFFF, 16);
                     }
                     // lwl
                     else if (opcode == 0x22) {
                         uint32 val = mem << ((rs & 3) * 8);
                         uint32 mask = uint32(0xFFFFFFFF) << ((rs & 3) * 8);
                         return (rt & ~mask) | val;
                     }
                     // lw
                     else if (opcode == 0x23) {
                         return mem;
                     }
                     // lbu
                     else if (opcode == 0x24) {
                         return (mem >> (24 - (rs & 3) * 8)) & 0xFF;
                     }
                     //  lhu
                     else if (opcode == 0x25) {
                         return (mem >> (16 - (rs & 2) * 8)) & 0xFFFF;
                     }
                     //  lwr
                     else if (opcode == 0x26) {
                         uint32 val = mem >> (24 - (rs & 3) * 8);
                         uint32 mask = uint32(0xFFFFFFFF) >> (24 - (rs & 3) * 8);
                         return (rt & ~mask) | val;
                     }
                     //  sb
                     else if (opcode == 0x28) {
                         uint32 val = (rt & 0xFF) << (24 - (rs & 3) * 8);
                         uint32 mask = 0xFFFFFFFF ^ uint32(0xFF << (24 - (rs & 3) * 8));
                         return (mem & mask) | val;
                     }
                     //  sh
                     else if (opcode == 0x29) {
                         uint32 val = (rt & 0xFFFF) << (16 - (rs & 2) * 8);
                         uint32 mask = 0xFFFFFFFF ^ uint32(0xFFFF << (16 - (rs & 2) * 8));
                         return (mem & mask) | val;
                     }
                     //  swl
                     else if (opcode == 0x2a) {
                         uint32 val = rt >> ((rs & 3) * 8);
                         uint32 mask = uint32(0xFFFFFFFF) >> ((rs & 3) * 8);
                         return (mem & ~mask) | val;
                     }
                     //  sw
                     else if (opcode == 0x2b) {
                         return rt;
                     }
                     //  swr
                     else if (opcode == 0x2e) {
                         uint32 val = rt << (24 - (rs & 3) * 8);
                         uint32 mask = uint32(0xFFFFFFFF) << (24 - (rs & 3) * 8);
                         return (mem & ~mask) | val;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;
                     }
                     // divu
                     else if (func == 0x1b) {
                         return rs;
                     }
                     // The rest includes transformed R-type arith imm instructions
                     // add
                     else if (func == 0x20) {
                         return (rs + rt);
                     }
                     // addu
                     else if (func == 0x21) {
                         return (rs + rt);
                     }
                     // sub
                     else if (func == 0x22) {
                         return (rs - rt);
                     }
                     // subu
                     else if (func == 0x23) {
                         return (rs - rt);
                     }
                     // and
                     else if (func == 0x24) {
                         return (rs & rt);
                     }
                     // or
                     else if (func == 0x25) {
                         return (rs | rt);
                     }
                     // xor
                     else if (func == 0x26) {
                         return (rs ^ rt);
                     }
                     // nor
                     else if (func == 0x27) {
                         return ~(rs | rt);
                     }
                     // slti
                     else if (func == 0x2a) {
                         return int32(rs) < int32(rt) ? 1 : 0;
                     }
                     // sltiu
                     else if (func == 0x2b) {
                         return rs < rt ? 1 : 0;
                     } else {
                         revert("invalid instruction");
                     }
                 } else {
                     // SPECIAL2
                     if (opcode == 0x1C) {
                         uint32 func = insn & 0x3f; // 6-bits
                         // mul
                         if (func == 0x2) {
                             return uint32(int32(rs) * int32(rt));
                         }
                         // clz, clo
                         else if (func == 0x20 || func == 0x21) {
                             if (func == 0x20) {
                                 rs = ~rs;
                             }
                             uint32 i = 0;
                             while (rs & 0x80000000 != 0) {
                                 i++;
                                 rs <<= 1;
                             }
                             return i;
                         }
                     }
                     // lui
                     else if (opcode == 0x0F) {
                         return rt << 16;
                     }
                     // lb
                     else if (opcode == 0x20) {
                         return SE((mem >> (24 - (rs & 3) * 8)) & 0xFF, 8);
                     }
                     // lh
                     else if (opcode == 0x21) {
                         return SE((mem >> (16 - (rs & 2) * 8)) & 0xFFFF, 16);
                     }
                     // lwl
                     else if (opcode == 0x22) {
                         uint32 val = mem << ((rs & 3) * 8);
                         uint32 mask = uint32(0xFFFFFFFF) << ((rs & 3) * 8);
                         return (rt & ~mask) | val;
                     }
                     // lw
                     else if (opcode == 0x23) {
                         return mem;
                     }
                     // lbu
                     else if (opcode == 0x24) {
                         return (mem >> (24 - (rs & 3) * 8)) & 0xFF;
                     }
                     //  lhu
                     else if (opcode == 0x25) {
                         return (mem >> (16 - (rs & 2) * 8)) & 0xFFFF;
                     }
                     //  lwr
                     else if (opcode == 0x26) {
                         uint32 val = mem >> (24 - (rs & 3) * 8);
                         uint32 mask = uint32(0xFFFFFFFF) >> (24 - (rs & 3) * 8);
                         return (rt & ~mask) | val;
                     }
                     //  sb
                     else if (opcode == 0x28) {
                         uint32 val = (rt & 0xFF) << (24 - (rs & 3) * 8);
                         uint32 mask = 0xFFFFFFFF ^ uint32(0xFF << (24 - (rs & 3) * 8));
                         return (mem & mask) | val;
                     }
                     //  sh
                     else if (opcode == 0x29) {
                         uint32 val = (rt & 0xFFFF) << (16 - (rs & 2) * 8);
                         uint32 mask = 0xFFFFFFFF ^ uint32(0xFFFF << (16 - (rs & 2) * 8));
                         return (mem & mask) | val;
                     }
                     //  swl
                     else if (opcode == 0x2a) {
                         uint32 val = rt >> ((rs & 3) * 8);
                         uint32 mask = uint32(0xFFFFFFFF) >> ((rs & 3) * 8);
                         return (mem & ~mask) | val;
                     }
                     //  sw
                     else if (opcode == 0x2b) {
                         return rt;
                     }
                     //  swr
                     else if (opcode == 0x2e) {
                         uint32 val = rt << (24 - (rs & 3) * 8);
                         uint32 mask = uint32(0xFFFFFFFF) << (24 - (rs & 3) * 8);
                         return (mem & ~mask) | val;
                     }
                     // ll
                     else if (opcode == 0x30) {
                         return mem;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;
                     }
                     // divu
                     else if (func == 0x1b) {
                         return rs;
                     }
                     // The rest includes transformed R-type arith imm instructions
                     // add
                     else if (func == 0x20) {
                         return (rs + rt);
                     }
                     // addu
                     else if (func == 0x21) {
                         return (rs + rt);
                     }
                     // sub
                     else if (func == 0x22) {
                         return (rs - rt);
                     }
                     // subu
                     else if (func == 0x23) {
                         return (rs - rt);
                     }
                     // and
                     else if (func == 0x24) {
                         return (rs & rt);
                     }
                     // or
                     else if (func == 0x25) {
                         return (rs | rt);
                     }
                     // xor
                     else if (func == 0x26) {
                         return (rs ^ rt);
                     }
                     // nor
                     else if (func == 0x27) {
                         return ~(rs | rt);
                     }
                     // slti
                     else if (func == 0x2a) {
                         return int32(rs) < int32(rt) ? 1 : 0;
                     }
                     // sltiu
                     else if (func == 0x2b) {
                         return rs < rt ? 1 : 0;
                     } else {
                         revert("invalid instruction");
                     }
                 } else {
                     // SPECIAL2
                     if (opcode == 0x1C) {
                         uint32 func = insn & 0x3f; // 6-bits
                         // mul
                         if (func == 0x2) {
                             return uint32(int32(rs) * int32(rt));
                         }
                         // clz, clo
                         else if (func == 0x20 || func == 0x21) {
                             if (func == 0x20) {
                                 rs = ~rs;
                             }
                             uint32 i = 0;
                             while (rs & 0x80000000 != 0) {
                                 i++;
                                 rs <<= 1;
                             }
                             return i;
                         }
                     }
                     // lui
                     else if (opcode == 0x0F) {
                         return rt << 16;
                     }
                     // lb
                     else if (opcode == 0x20) {
                         return SE((mem >> (24 - (rs & 3) * 8)) & 0xFF, 8);
                     }
                     // lh
                     else if (opcode == 0x21) {
                         return SE((mem >> (16 - (rs & 2) * 8)) & 0xFFFF, 16);
                     }
                     // lwl
                     else if (opcode == 0x22) {
                         uint32 val = mem << ((rs & 3) * 8);
                         uint32 mask = uint32(0xFFFFFFFF) << ((rs & 3) * 8);
                         return (rt & ~mask) | val;
                     }
                     // lw
                     else if (opcode == 0x23) {
                         return mem;
                     }
                     // lbu
                     else if (opcode == 0x24) {
                         return (mem >> (24 - (rs & 3) * 8)) & 0xFF;
                     }
                     //  lhu
                     else if (opcode == 0x25) {
                         return (mem >> (16 - (rs & 2) * 8)) & 0xFFFF;
                     }
                     //  lwr
                     else if (opcode == 0x26) {
                         uint32 val = mem >> (24 - (rs & 3) * 8);
                         uint32 mask = uint32(0xFFFFFFFF) >> (24 - (rs & 3) * 8);
                         return (rt & ~mask) | val;
                     }
                     //  sb
                     else if (opcode == 0x28) {
                         uint32 val = (rt & 0xFF) << (24 - (rs & 3) * 8);
                         uint32 mask = 0xFFFFFFFF ^ uint32(0xFF << (24 - (rs & 3) * 8));
                         return (mem & mask) | val;
                     }
                     //  sh
                     else if (opcode == 0x29) {
                         uint32 val = (rt & 0xFFFF) << (16 - (rs & 2) * 8);
                         uint32 mask = 0xFFFFFFFF ^ uint32(0xFFFF << (16 - (rs & 2) * 8));
                         return (mem & mask) | val;
                     }
                     //  swl
                     else if (opcode == 0x2a) {
                         uint32 val = rt >> ((rs & 3) * 8);
                         uint32 mask = uint32(0xFFFFFFFF) >> ((rs & 3) * 8);
                         return (mem & ~mask) | val;
                     }
                     //  sw
                     else if (opcode == 0x2b) {
                         return rt;
                     }
                     //  swr
                     else if (opcode == 0x2e) {
                         uint32 val = rt << (24 - (rs & 3) * 8);
                         uint32 mask = uint32(0xFFFFFFFF) << (24 - (rs & 3) * 8);
                         return (mem & ~mask) | val;
                     }
                     // ll
                     else if (opcode == 0x30) {
                         return mem;
                     }
                     // sc
                     else if (opcode == 0x38) {
                         return rt;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;
                     }
                     // divu
                     else if (func == 0x1b) {
                         return rs;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;
                     }
                     // divu
                     else if (func == 0x1b) {
                         return rs;
                     }
                     // The rest includes transformed R-type arith imm instructions
                     // add
                     else if (func == 0x20) {
                         return (rs + rt);

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;
                     }
                     // divu
                     else if (func == 0x1b) {
                         return rs;
                     }
                     // The rest includes transformed R-type arith imm instructions
                     // add
                     else if (func == 0x20) {
                         return (rs + rt);
                     }
                     // addu
                     else if (func == 0x21) {
                         return (rs + rt);

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;
                     }
                     // divu
                     else if (func == 0x1b) {
                         return rs;
                     }
                     // The rest includes transformed R-type arith imm instructions
                     // add
                     else if (func == 0x20) {
                         return (rs + rt);
                     }
                     // addu
                     else if (func == 0x21) {
                         return (rs + rt);
                     }
                     // sub
                     else if (func == 0x22) {
                         return (rs - rt);

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;
                     }
                     // divu
                     else if (func == 0x1b) {
                         return rs;
                     }
                     // The rest includes transformed R-type arith imm instructions
                     // add
                     else if (func == 0x20) {
                         return (rs + rt);
                     }
                     // addu
                     else if (func == 0x21) {
                         return (rs + rt);
                     }
                     // sub
                     else if (func == 0x22) {
                         return (rs - rt);
                     }
                     // subu
                     else if (func == 0x23) {
                         return (rs - rt);

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;
                     }
                     // divu
                     else if (func == 0x1b) {
                         return rs;
                     }
                     // The rest includes transformed R-type arith imm instructions
                     // add
                     else if (func == 0x20) {
                         return (rs + rt);
                     }
                     // addu
                     else if (func == 0x21) {
                         return (rs + rt);
                     }
                     // sub
                     else if (func == 0x22) {
                         return (rs - rt);
                     }
                     // subu
                     else if (func == 0x23) {
                         return (rs - rt);
                     }
                     // and
                     else if (func == 0x24) {
                         return (rs & rt);

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;
                     }
                     // divu
                     else if (func == 0x1b) {
                         return rs;
                     }
                     // The rest includes transformed R-type arith imm instructions
                     // add
                     else if (func == 0x20) {
                         return (rs + rt);
                     }
                     // addu
                     else if (func == 0x21) {
                         return (rs + rt);
                     }
                     // sub
                     else if (func == 0x22) {
                         return (rs - rt);
                     }
                     // subu
                     else if (func == 0x23) {
                         return (rs - rt);
                     }
                     // and
                     else if (func == 0x24) {
                         return (rs & rt);
                     }
                     // or
                     else if (func == 0x25) {
                         return (rs | rt);

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;
                     }
                     // divu
                     else if (func == 0x1b) {
                         return rs;
                     }
                     // The rest includes transformed R-type arith imm instructions
                     // add
                     else if (func == 0x20) {
                         return (rs + rt);
                     }
                     // addu
                     else if (func == 0x21) {
                         return (rs + rt);
                     }
                     // sub
                     else if (func == 0x22) {
                         return (rs - rt);
                     }
                     // subu
                     else if (func == 0x23) {
                         return (rs - rt);
                     }
                     // and
                     else if (func == 0x24) {
                         return (rs & rt);
                     }
                     // or
                     else if (func == 0x25) {
                         return (rs | rt);
                     }
                     // xor
                     else if (func == 0x26) {
                         return (rs ^ rt);

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;
                     }
                     // divu
                     else if (func == 0x1b) {
                         return rs;
                     }
                     // The rest includes transformed R-type arith imm instructions
                     // add
                     else if (func == 0x20) {
                         return (rs + rt);
                     }
                     // addu
                     else if (func == 0x21) {
                         return (rs + rt);
                     }
                     // sub
                     else if (func == 0x22) {
                         return (rs - rt);
                     }
                     // subu
                     else if (func == 0x23) {
                         return (rs - rt);
                     }
                     // and
                     else if (func == 0x24) {
                         return (rs & rt);
                     }
                     // or
                     else if (func == 0x25) {
                         return (rs | rt);
                     }
                     // xor
                     else if (func == 0x26) {
                         return (rs ^ rt);
                     }
                     // nor
                     else if (func == 0x27) {
                         return ~(rs | rt);

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;
                     }
                     // divu
                     else if (func == 0x1b) {
                         return rs;
                     }
                     // The rest includes transformed R-type arith imm instructions
                     // add
                     else if (func == 0x20) {
                         return (rs + rt);
                     }
                     // addu
                     else if (func == 0x21) {
                         return (rs + rt);
                     }
                     // sub
                     else if (func == 0x22) {
                         return (rs - rt);
                     }
                     // subu
                     else if (func == 0x23) {
                         return (rs - rt);
                     }
                     // and
                     else if (func == 0x24) {
                         return (rs & rt);
                     }
                     // or
                     else if (func == 0x25) {
                         return (rs | rt);
                     }
                     // xor
                     else if (func == 0x26) {
                         return (rs ^ rt);
                     }
                     // nor
                     else if (func == 0x27) {
                         return ~(rs | rt);
                     }
                     // slti
                     else if (func == 0x2a) {
                         return int32(rs) < int32(rt) ? 1 : 0;

808:     /// @notice Execute an instruction.
         function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {
             unchecked {
                 uint32 opcode = insn >> 26; // 6-bits
     
                 if (opcode == 0 || (opcode >= 8 && opcode < 0xF)) {
                     uint32 func = insn & 0x3f; // 6-bits
                     assembly {
                         // transform ArithLogI to SPECIAL
                         switch opcode
                         // addi
                         case 0x8 { func := 0x20 }
                         // addiu
                         case 0x9 { func := 0x21 }
                         // stli
                         case 0xA { func := 0x2A }
                         // sltiu
                         case 0xB { func := 0x2B }
                         // andi
                         case 0xC { func := 0x24 }
                         // ori
                         case 0xD { func := 0x25 }
                         // xori
                         case 0xE { func := 0x26 }
                     }
     
                     // sll
                     if (func == 0x00) {
                         return rt << ((insn >> 6) & 0x1F);
                     }
                     // srl
                     else if (func == 0x02) {
                         return rt >> ((insn >> 6) & 0x1F);
                     }
                     // sra
                     else if (func == 0x03) {
                         uint32 shamt = (insn >> 6) & 0x1F;
                         return SE(rt >> shamt, 32 - shamt);
                     }
                     // sllv
                     else if (func == 0x04) {
                         return rt << (rs & 0x1F);
                     }
                     // srlv
                     else if (func == 0x6) {
                         return rt >> (rs & 0x1F);
                     }
                     // srav
                     else if (func == 0x07) {
                         return SE(rt >> rs, 32 - rs);
                     }
                     // functs in range [0x8, 0x1b] are handled specially by other functions
                     // Explicitly enumerate each funct in range to reduce code diff against Go Vm
                     // jr
                     else if (func == 0x08) {
                         return rs;
                     }
                     // jalr
                     else if (func == 0x09) {
                         return rs;
                     }
                     // movz
                     else if (func == 0x0a) {
                         return rs;
                     }
                     // movn
                     else if (func == 0x0b) {
                         return rs;
                     }
                     // syscall
                     else if (func == 0x0c) {
                         return rs;
                     }
                     // 0x0d - break not supported
                     // sync
                     else if (func == 0x0f) {
                         return rs;
                     }
                     // mfhi
                     else if (func == 0x10) {
                         return rs;
                     }
                     // mthi
                     else if (func == 0x11) {
                         return rs;
                     }
                     // mflo
                     else if (func == 0x12) {
                         return rs;
                     }
                     // mtlo
                     else if (func == 0x13) {
                         return rs;
                     }
                     // mult
                     else if (func == 0x18) {
                         return rs;
                     }
                     // multu
                     else if (func == 0x19) {
                         return rs;
                     }
                     // div
                     else if (func == 0x1a) {
                         return rs;
                     }
                     // divu
                     else if (func == 0x1b) {
                         return rs;
                     }
                     // The rest includes transformed R-type arith imm instructions
                     // add
                     else if (func == 0x20) {
                         return (rs + rt);
                     }
                     // addu
                     else if (func == 0x21) {
                         return (rs + rt);
                     }
                     // sub
                     else if (func == 0x22) {
                         return (rs - rt);
                     }
                     // subu
                     else if (func == 0x23) {
                         return (rs - rt);
                     }
                     // and
                     else if (func == 0x24) {
                         return (rs & rt);
                     }
                     // or
                     else if (func == 0x25) {
                         return (rs | rt);
                     }
                     // xor
                     else if (func == 0x26) {
                         return (rs ^ rt);
                     }
                     // nor
                     else if (func == 0x27) {
                         return ~(rs | rt);
                     }
                     // slti
                     else if (func == 0x2a) {
                         return int32(rs) < int32(rt) ? 1 : 0;
                     }
                     // sltiu
                     else if (func == 0x2b) {
                         return rs < rt ? 1 : 0;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/MIPS.sol)

```solidity
File: src/dispute/DisputeGameFactory.sol

157:         // If the `_start` index is greater than or equal to the game array length or `_n == 0`, return an empty array.
             if (_start >= _disputeGameList.length || _n == 0) return games_;
     
             // Allocate enough memory for the full array, but start the array's length at `0`. We may not use all of the
             // memory allocated, but we don't know ahead of time the final size of the array.
             assembly {
                 games_ := mload(0x40)
                 mstore(0x40, add(games_, add(0x20, shl(0x05, _n))))
             }

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol)

### <a name="NC-18"></a>[NC-18] Take advantage of Custom Error's return value property
An important feature of Custom Error is that values such as address, tokenID, msg.value can be written inside the () sign, this kind of approach provides a serious advantage in debugging and examining the revert details of dapps such as tenderly.

*Instances (64)*:
```solidity
File: src/cannon/PreimageOracle.sol

135:             revert PartOffsetOOB();

419:         if (msg.value < MIN_BOND_SIZE) revert InsufficientBond();

422:         if (msg.sender != tx.origin) revert NotEOA();

425:         if (_partOffset >= _claimedSize + 8) revert PartOffsetOOB();

428:         if (_claimedSize < MIN_LPP_SIZE_BYTES) revert InvalidInputSize();

465:         if (msg.sender != tx.origin) revert NotEOA();

468:         if (metaData.claimedSize() == 0) revert NotInitialized();

471:         if (metaData.timestamp() != 0) revert AlreadyFinalized();

475:         if (blocksProcessed != _inputStartBlock) revert WrongStartingBlock();

535:         if (blocksProcessed > MAX_LEAF_COUNT) revert TreeSizeOverflow();

547:             if (metaData.bytesProcessed() != metaData.claimedSize()) revert InvalidInputSize();

586:         ) revert InvalidProof();

589:         if (keccak256(abi.encode(_stateMatrix)) != _preState.stateCommitment) revert InvalidPreimage();

592:         if (_preState.index + 1 != _postState.index) revert StatesNotContiguous();

599:         if (keccak256(abi.encode(_stateMatrix)) == _postState.stateCommitment) revert PostStateMatches();

619:         if (!_verify(_postStateProof, root, _postState.index, _hashLeaf(_postState))) revert InvalidProof();

622:         if (_postState.index != 0) revert StatesNotContiguous();

630:         if (keccak256(abi.encode(stateMatrix)) == _postState.stateCommitment) revert PostStateMatches();

654:         if (metaData.countered()) revert BadProposal();

657:         if (block.timestamp - metaData.timestamp() <= CHALLENGE_PERIOD) revert ActiveProposal();

666:         ) revert InvalidProof();

669:         if (keccak256(abi.encode(_stateMatrix)) != _preState.stateCommitment) revert InvalidPreimage();

673:             revert StatesNotContiguous();

742:             if (relativeOffset + 32 >= _input.length && !_finalize) revert PartOffsetOOB();

793:         if (!success) revert BondTransferFailed();

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

```solidity
File: src/dispute/DisputeGameFactory.sol

100:         if (msg.value != initBonds[_gameType]) revert IncorrectBondAmount();

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

138:         if (_maxGameDepth > LibPosition.MAX_POSITION_BITLEN - 1) revert MaxDepthTooLarge();

140:         if (_splitDepth >= _maxGameDepth) revert InvalidSplitDepth();

142:         if (_clockExtension.raw() > _maxClockDuration.raw()) revert InvalidClockExtension();

170:         if (initialized) revert AlreadyInitialized();

176:         if (root.raw() == bytes32(0)) revert AnchorRootNotFound();

204:         if (l2BlockNumber() <= rootBlockNumber) revert UnexpectedRootClaim(rootClaim());

244:         if (status != GameStatus.IN_PROGRESS) revert GameNotInProgress();

255:         if (stepPos.depth() != MAX_GAME_DEPTH + 1) revert InvalidParent();

285:         if (keccak256(_stateData) << 8 != preStateClaim.raw() << 8) revert InvalidPrestate();

304:         if (parentPostAgree == validStep) revert ValidStep();

307:         if (parent.counteredBy != address(0)) revert DuplicateStep();

321:         if (status != GameStatus.IN_PROGRESS) revert GameNotInProgress();

327:         if (Claim.unwrap(parent.claim) != Claim.unwrap(_disputed)) revert InvalidDisputedClaimIndex();

340:             revert CannotDefendRootClaim();

345:         if (l2BlockNumberChallenged && _challengeIndex == 0) revert L2BlockNumberChallenged();

351:         if (nextPositionDepth > MAX_GAME_DEPTH) revert GameDepthExceeded();

360:         if (getRequiredBond(nextPosition) != msg.value) revert IncorrectBondAmount();

369:         if (nextDuration.raw() == MAX_CLOCK_DURATION.raw()) revert ClockTimeExceeded();

391:         if (claims[claimHash]) revert ClaimAlreadyExists();

431:         if (status != GameStatus.IN_PROGRESS) revert GameNotInProgress();

460:             revert InvalidLocalIdent();

499:         if (status != GameStatus.IN_PROGRESS) revert GameNotInProgress();

502:         if (l2BlockNumberChallenged) revert L2BlockNumberChallenged();

505:         if (Hashing.hashOutputRootProof(_outputRootProof) != rootClaim().raw()) revert InvalidOutputRootProof();

508:         if (keccak256(_headerRLP) != _outputRootProof.latestBlockhash) revert InvalidHeaderRLP();

516:         if (rawBlockNumber.length > 32) revert InvalidHeaderRLP();

528:         if (blockNumber == l2BlockNumber()) revert BlockNumberMatches();

543:         if (status != GameStatus.IN_PROGRESS) revert GameNotInProgress();

546:         if (!resolvedSubgames[0]) revert OutOfOrderResolution();

562:         if (status != GameStatus.IN_PROGRESS) revert GameNotInProgress();

570:         if (challengeClockDuration.raw() < MAX_CLOCK_DURATION.raw()) revert ClockNotExpired();

573:         if (resolvedSubgames[_claimIndex]) revert ClaimAlreadyResolved();

611:             if (!resolvedSubgames[challengeIndex]) revert OutOfOrderResolution();

704:         if (depth > MAX_GAME_DEPTH) revert GameDepthExceeded();

753:         if (recipientCredit == 0) revert NoCreditToClaim();

760:         if (!success) revert BondTransferFailed();

770:             revert GameNotInProgress();

941:         if (claim.position.depth() <= SPLIT_DEPTH) revert ClaimAboveSplit();

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="NC-19"></a>[NC-19] Contract does not follow the Solidity style guide's suggested layout ordering
The [style guide](https://docs.soliditylang.org/en/v0.8.16/style-guide.html#order-of-layout) says that, within a contract, the ordering should be:

1) Type declarations
2) State variables
3) Events
4) Modifiers
5) Functions

However, the contract(s) below do not follow this ordering

*Instances (2)*:
```solidity
File: src/cannon/MIPS.sol

1: 
   Current order:
   StructDefinition.State
   VariableDeclaration.BRK_START
   VariableDeclaration.version
   VariableDeclaration.FD_STDIN
   VariableDeclaration.FD_STDOUT
   VariableDeclaration.FD_STDERR
   VariableDeclaration.FD_HINT_READ
   VariableDeclaration.FD_HINT_WRITE
   VariableDeclaration.FD_PREIMAGE_READ
   VariableDeclaration.FD_PREIMAGE_WRITE
   VariableDeclaration.EBADF
   VariableDeclaration.EINVAL
   VariableDeclaration.ORACLE
   FunctionDefinition.constructor
   FunctionDefinition.oracle
   FunctionDefinition.SE
   FunctionDefinition.outputState
   FunctionDefinition.handleSyscall
   FunctionDefinition.handleBranch
   FunctionDefinition.handleHiLo
   FunctionDefinition.handleJump
   FunctionDefinition.handleRd
   FunctionDefinition.proofOffset
   FunctionDefinition.readMem
   FunctionDefinition.writeMem
   FunctionDefinition.step
   FunctionDefinition.execute
   
   Suggested order:
   VariableDeclaration.BRK_START
   VariableDeclaration.version
   VariableDeclaration.FD_STDIN
   VariableDeclaration.FD_STDOUT
   VariableDeclaration.FD_STDERR
   VariableDeclaration.FD_HINT_READ
   VariableDeclaration.FD_HINT_WRITE
   VariableDeclaration.FD_PREIMAGE_READ
   VariableDeclaration.FD_PREIMAGE_WRITE
   VariableDeclaration.EBADF
   VariableDeclaration.EINVAL
   VariableDeclaration.ORACLE
   StructDefinition.State
   FunctionDefinition.constructor
   FunctionDefinition.oracle
   FunctionDefinition.SE
   FunctionDefinition.outputState
   FunctionDefinition.handleSyscall
   FunctionDefinition.handleBranch
   FunctionDefinition.handleHiLo
   FunctionDefinition.handleJump
   FunctionDefinition.handleRd
   FunctionDefinition.proofOffset
   FunctionDefinition.readMem
   FunctionDefinition.writeMem
   FunctionDefinition.step
   FunctionDefinition.execute

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/MIPS.sol)

```solidity
File: src/cannon/PreimageOracle.sol

1: 
   Current order:
   VariableDeclaration.CHALLENGE_PERIOD
   VariableDeclaration.MIN_LPP_SIZE_BYTES
   VariableDeclaration.MIN_BOND_SIZE
   VariableDeclaration.KECCAK_TREE_DEPTH
   VariableDeclaration.MAX_LEAF_COUNT
   VariableDeclaration.version
   VariableDeclaration.preimageLengths
   VariableDeclaration.preimageParts
   VariableDeclaration.preimagePartOk
   StructDefinition.Leaf
   StructDefinition.LargePreimageProposalKeys
   VariableDeclaration.zeroHashes
   VariableDeclaration.proposals
   VariableDeclaration.proposalBranches
   VariableDeclaration.proposalMetadata
   VariableDeclaration.proposalBonds
   VariableDeclaration.proposalParts
   VariableDeclaration.proposalBlocks
   FunctionDefinition.constructor
   FunctionDefinition.readPreimage
   FunctionDefinition.loadLocalData
   FunctionDefinition.loadKeccak256PreimagePart
   FunctionDefinition.loadSha256PreimagePart
   FunctionDefinition.loadBlobPreimagePart
   FunctionDefinition.loadPrecompilePreimagePart
   FunctionDefinition.proposalCount
   FunctionDefinition.proposalBlocksLen
   FunctionDefinition.challengePeriod
   FunctionDefinition.minProposalSize
   FunctionDefinition.initLPP
   FunctionDefinition.addLeavesLPP
   FunctionDefinition.challengeLPP
   FunctionDefinition.challengeFirstLPP
   FunctionDefinition.squeezeLPP
   FunctionDefinition.getTreeRootLPP
   FunctionDefinition._extractPreimagePart
   FunctionDefinition._verify
   FunctionDefinition._payoutBond
   FunctionDefinition._hashLeaf
   
   Suggested order:
   VariableDeclaration.CHALLENGE_PERIOD
   VariableDeclaration.MIN_LPP_SIZE_BYTES
   VariableDeclaration.MIN_BOND_SIZE
   VariableDeclaration.KECCAK_TREE_DEPTH
   VariableDeclaration.MAX_LEAF_COUNT
   VariableDeclaration.version
   VariableDeclaration.preimageLengths
   VariableDeclaration.preimageParts
   VariableDeclaration.preimagePartOk
   VariableDeclaration.zeroHashes
   VariableDeclaration.proposals
   VariableDeclaration.proposalBranches
   VariableDeclaration.proposalMetadata
   VariableDeclaration.proposalBonds
   VariableDeclaration.proposalParts
   VariableDeclaration.proposalBlocks
   StructDefinition.Leaf
   StructDefinition.LargePreimageProposalKeys
   FunctionDefinition.constructor
   FunctionDefinition.readPreimage
   FunctionDefinition.loadLocalData
   FunctionDefinition.loadKeccak256PreimagePart
   FunctionDefinition.loadSha256PreimagePart
   FunctionDefinition.loadBlobPreimagePart
   FunctionDefinition.loadPrecompilePreimagePart
   FunctionDefinition.proposalCount
   FunctionDefinition.proposalBlocksLen
   FunctionDefinition.challengePeriod
   FunctionDefinition.minProposalSize
   FunctionDefinition.initLPP
   FunctionDefinition.addLeavesLPP
   FunctionDefinition.challengeLPP
   FunctionDefinition.challengeFirstLPP
   FunctionDefinition.squeezeLPP
   FunctionDefinition.getTreeRootLPP
   FunctionDefinition._extractPreimagePart
   FunctionDefinition._verify
   FunctionDefinition._payoutBond
   FunctionDefinition._hashLeaf

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

### <a name="NC-20"></a>[NC-20] Use Underscores for Number Literals (add an underscore every 3 digits)

*Instances (9)*:
```solidity
File: src/cannon/MIPS.sol

170:             if (syscall_no == 4090) {

172:                 if (sz & 4095 != 0) {

174:                     sz += 4096 - (sz & 4095);

184:             else if (syscall_no == 4045) {

188:             else if (syscall_no == 4120) {

192:             else if (syscall_no == 4246) {

198:             else if (syscall_no == 4003) {

248:             else if (syscall_no == 4004) {

282:             else if (syscall_no == 4055) {

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/MIPS.sol)

### <a name="NC-21"></a>[NC-21] Internal and private variables and functions names should begin with an underscore
According to the Solidity Style Guide, Non-`external` variable and function names should begin with an [underscore](https://docs.soliditylang.org/en/latest/style-guide.html#underscore-prefix-for-non-external-functions-and-variables)

*Instances (26)*:
```solidity
File: src/cannon/MIPS.sol

75:     function SE(uint32 _dat, uint32 _idx) internal pure returns (uint32 out_) {

86:     function outputState() internal returns (bytes32 out_) {

151:     function handleSyscall(bytes32 _localContext) internal returns (bytes32 out_) {

319:     function handleBranch(uint32 _opcode, uint32 _insn, uint32 _rtReg, uint32 _rs) internal returns (bytes32 out_) {

383:     function handleHiLo(uint32 _func, uint32 _rs, uint32 _rt, uint32 _storeReg) internal returns (bytes32 out_) {

460:     function handleJump(uint32 _linkReg, uint32 _dest) internal returns (bytes32 out_) {

492:     function handleRd(uint32 _storeReg, uint32 _val, bool _conditional) internal returns (bytes32 out_) {

520:     function proofOffset(uint8 _proofIndex) internal pure returns (uint256 offset_) {

538:     function readMem(uint32 _addr, uint8 _proofIndex) internal pure returns (uint32 out_) {

592:     function writeMem(uint32 _addr, uint8 _proofIndex, uint32 _val) internal pure {

809:     function execute(uint32 insn, uint32 rs, uint32 rt, uint32 mem) internal pure returns (uint32 out) {

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/MIPS.sol)

```solidity
File: src/cannon/PreimageKeyLib.sol

12:     function localizeIdent(uint256 _ident, bytes32 _localContext) internal view returns (bytes32 key_) {

29:     function localize(bytes32 _key, bytes32 _localContext) internal view returns (bytes32 localizedKey_) {

47:     function keccak256PreimageKey(bytes memory _preimage) internal pure returns (bytes32 key_) {

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageKeyLib.sol)

```solidity
File: src/cannon/libraries/CannonTypes.sol

32:             self_ := or(shl(160, _partOffset), and(_self, not(shl(160, U32_MASK))))

38:             self_ := or(shl(128, _claimedSize), and(_self, not(shl(128, U32_MASK))))

44:             self_ := or(shl(96, _blocksProcessed), and(_self, not(shl(96, U32_MASK))))

50:             self_ := or(shl(64, _bytesProcessed), and(_self, not(shl(64, U32_MASK))))

56:             self_ := or(_countered, and(_self, not(U64_MASK)))

66:     function partOffset(LPPMetaData _self) internal pure returns (uint64 partOffset_) {

72:     function claimedSize(LPPMetaData _self) internal pure returns (uint32 claimedSize_) {

78:     function blocksProcessed(LPPMetaData _self) internal pure returns (uint32 blocksProcessed_) {

84:     function bytesProcessed(LPPMetaData _self) internal pure returns (uint32 bytesProcessed_) {

90:     function countered(LPPMetaData _self) internal pure returns (bool countered_) {

96: 

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/libraries/CannonTypes.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

85:     bool internal initialized;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="NC-22"></a>[NC-22] Constants should be defined rather than using magic numbers

*Instances (41)*:
```solidity
File: src/cannon/MIPS.sol

144:             out_ := or(and(not(shl(248, 0xFF)), out_), shl(248, status))

524:             offset_ = 420 + (uint256(_proofIndex) * (28 * 32));

988:                     return SE((mem >> (24 - (rs & 3) * 8)) & 0xFF, 8);

992:                     return SE((mem >> (16 - (rs & 2) * 8)) & 0xFFFF, 16);

1006:                     return (mem >> (24 - (rs & 3) * 8)) & 0xFF;

1010:                     return (mem >> (16 - (rs & 2) * 8)) & 0xFFFF;

1014:                     uint32 val = mem >> (24 - (rs & 3) * 8);

1015:                     uint32 mask = uint32(0xFFFFFFFF) >> (24 - (rs & 3) * 8);

1020:                     uint32 val = (rt & 0xFF) << (24 - (rs & 3) * 8);

1021:                     uint32 mask = 0xFFFFFFFF ^ uint32(0xFF << (24 - (rs & 3) * 8));

1026:                     uint32 val = (rt & 0xFFFF) << (16 - (rs & 2) * 8);

1027:                     uint32 mask = 0xFFFFFFFF ^ uint32(0xFFFF << (16 - (rs & 2) * 8));

1042:                     uint32 val = rt << (24 - (rs & 3) * 8);

1043:                     uint32 mask = uint32(0xFFFFFFFF) << (24 - (rs & 3) * 8);

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/MIPS.sol)

```solidity
File: src/cannon/PreimageKeyLib.sol

16:             key_ := or(shl(248, 1), and(_ident, not(shl(248, 0xFF))))

38:             localizedKey_ := or(and(keccak256(0, 0x60), not(shl(248, 0xFF))), shl(248, 1))

56:             key_ := or(and(h, not(shl(248, 0xFF))), shl(248, 2))

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageKeyLib.sol)

```solidity
File: src/cannon/PreimageOracle.sol

145:             mstore(0x00, shl(192, _size))

178:             mstore(ptr, shl(192, size))

187:             key := or(and(h, not(shl(248, 0xFF))), shl(248, 0x02))

214:             mstore(ptr, shl(192, size))

236:             key := or(and(h, not(shl(248, 0xFF))), shl(248, 4))

271:             let versionedHash := or(and(mload(0x00), not(shl(248, 0xFF))), shl(248, 0x01))

313:             mstore(ptr, shl(192, 0x20))

327:             key := or(and(h, not(shl(248, 0xFF))), shl(248, 0x05))

347:             calldatacopy(add(20, ptr), _input.offset, _input.length)

349:             let h := keccak256(ptr, add(20, _input.length))

351:             key := or(and(h, not(shl(248, 0xFF))), shl(248, 0x06))

358:                     add(20, ptr), // input ptr

375:             mstore(ptr, shl(192, size))

561:             mstore(0x00, shl(96, caller()))

682:             finalDigest := or(and(finalDigest, not(shl(248, 0xFF))), shl(248, 0x02))

730:                 mstore(0x00, shl(192, claimedSize))

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

```solidity
File: src/cannon/libraries/CannonTypes.sol

26:             self_ := or(shl(192, _timestamp), and(_self, not(shl(192, U64_MASK))))

32:             self_ := or(shl(160, _partOffset), and(_self, not(shl(160, U32_MASK))))

44:             self_ := or(shl(96, _blocksProcessed), and(_self, not(shl(96, U32_MASK))))

50:             self_ := or(shl(64, _bytesProcessed), and(_self, not(shl(64, U32_MASK))))

62:             timestamp_ := shr(192, _self)

68:             partOffset_ := and(shr(160, _self), U32_MASK)

80:             blocksProcessed_ := and(shr(96, _self), U32_MASK)

86:             bytesProcessed_ := and(shr(64, _self), U32_MASK)

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/libraries/CannonTypes.sol)

### <a name="NC-23"></a>[NC-23] `public` functions not called by the contract should be declared `external` instead

*Instances (2)*:
```solidity
File: src/cannon/MIPS.sol

640:     function step(bytes calldata _stateData, bytes calldata _proof, bytes32 _localContext) public returns (bytes32) {

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/MIPS.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

465:     function getNumToResolve(uint256 _claimIndex) public view returns (uint256 numRemainingChildren_) {

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="NC-24"></a>[NC-24] Variables need not be initialized to zero
The default value for variables is zero, so initializing them to zero is superfluous.

*Instances (6)*:
```solidity
File: src/cannon/MIPS.sol

161:             uint32 v0 = 0;

162:             uint32 v1 = 0;

525:             uint256 s = 0;

974:                         uint32 i = 0;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/MIPS.sol)

```solidity
File: src/cannon/PreimageOracle.sol

94:         for (uint256 height = 0; height < KECCAK_TREE_DEPTH - 1; height++) {

698:         for (uint256 height = 0; height < KECCAK_TREE_DEPTH; height++) {

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)


## Low Issues


| |Issue|Instances|
|-|:-|:-:|
| [L-1](#L-1) | Use of `tx.origin` is unsafe in almost every context | 2 |
| [L-2](#L-2) | Use a 2-step ownership transfer pattern | 1 |
| [L-3](#L-3) | `abi.encodePacked()` should not be used with dynamic types when passing the result to a hash function such as `keccak256()` | 2 |
| [L-4](#L-4) | Use of `tx.origin` is unsafe in almost every context | 2 |
| [L-5](#L-5) | Do not leave an implementation contract uninitialized | 2 |
| [L-6](#L-6) | Division by zero not prevented | 3 |
| [L-7](#L-7) | Duplicate import statements | 2 |
| [L-8](#L-8) | External call recipient may consume all transaction gas | 2 |
| [L-9](#L-9) | Initializers could be front-run | 4 |
| [L-10](#L-10) | Solidity version 0.8.20+ may not work on other chains due to `PUSH0` | 6 |
| [L-11](#L-11) | Use `Ownable2Step.transferOwnership` instead of `Ownable.transferOwnership` | 2 |
| [L-12](#L-12) | Consider using OpenZeppelin's SafeCast library to prevent unexpected overflows when downcasting | 10 |
| [L-13](#L-13) | Upgradeable contract is missing a `__gap[50]` storage variable to allow for new storage variables in later versions | 3 |
| [L-14](#L-14) | Upgradeable contract not initialized | 12 |
### <a name="L-1"></a>[L-1] Use of `tx.origin` is unsafe in almost every context
According to [Vitalik Buterin](https://ethereum.stackexchange.com/questions/196/how-do-i-make-my-dapp-serenity-proof), contracts should _not_ `assume that tx.origin will continue to be usable or meaningful`. An example of this is [EIP-3074](https://eips.ethereum.org/EIPS/eip-3074#allowing-txorigin-as-signer-1) which explicitly mentions the intention to change its semantics when it's used with new op codes. There have also been calls to [remove](https://github.com/ethereum/solidity/issues/683) `tx.origin`, and there are [security issues](solidity.readthedocs.io/en/v0.4.24/security-considerations.html#tx-origin) associated with using it for authorization. For these reasons, it's best to completely avoid the feature.

*Instances (2)*:
```solidity
File: src/cannon/PreimageOracle.sol

422:         if (msg.sender != tx.origin) revert NotEOA();

465:         if (msg.sender != tx.origin) revert NotEOA();

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

### <a name="L-2"></a>[L-2] Use a 2-step ownership transfer pattern
Recommend considering implementing a two step process where the owner or admin nominates an account and the nominated account needs to call an `acceptOwnership()` function for the transfer of ownership to fully succeed. This ensures the nominated EOA account is a valid and active account. Lack of two-step procedure for critical operations leaves them error-prone. Consider adding two step procedure on the critical functions.

*Instances (1)*:
```solidity
File: src/dispute/DisputeGameFactory.sol

19: contract DisputeGameFactory is OwnableUpgradeable, IDisputeGameFactory, ISemver {

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol)

### <a name="L-3"></a>[L-3] `abi.encodePacked()` should not be used with dynamic types when passing the result to a hash function such as `keccak256()`
Use `abi.encode()` instead which will pad items to 32 bytes, which will [prevent hash collisions](https://docs.soliditylang.org/en/v0.8.13/abi-spec.html#non-standard-packed-mode) (e.g. `abi.encodePacked(0x123,0x456)` => `0x123456` => `abi.encodePacked(0x1,0x23456)`, but `abi.encode(0x123,0x456)` => `0x0...1230...456`). "Unless there is a compelling reason, `abi.encode` should be preferred". If there is only one argument to `abi.encodePacked()` it can often be cast to `bytes()` or `bytes32()` [instead](https://ethereum.stackexchange.com/questions/30912/how-to-compare-strings-in-solidity#answer-82739).
If all arguments are strings and or bytes, `bytes.concat()` should be used instead

*Instances (2)*:
```solidity
File: src/cannon/PreimageOracle.sol

798:         leaf_ = keccak256(abi.encodePacked(_leaf.input, _leaf.index, _leaf.stateCommitment));

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

```solidity
File: src/dispute/DisputeGameFactory.sol

123:         if (GameId.unwrap(_disputeGames[uuid]) != bytes32(0)) revert GameAlreadyExists(uuid);

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol)

### <a name="L-4"></a>[L-4] Use of `tx.origin` is unsafe in almost every context
According to [Vitalik Buterin](https://ethereum.stackexchange.com/questions/196/how-do-i-make-my-dapp-serenity-proof), contracts should _not_ `assume that tx.origin will continue to be usable or meaningful`. An example of this is [EIP-3074](https://eips.ethereum.org/EIPS/eip-3074#allowing-txorigin-as-signer-1) which explicitly mentions the intention to change its semantics when it's used with new op codes. There have also been calls to [remove](https://github.com/ethereum/solidity/issues/683) `tx.origin`, and there are [security issues](solidity.readthedocs.io/en/v0.4.24/security-considerations.html#tx-origin) associated with using it for authorization. For these reasons, it's best to completely avoid the feature.

*Instances (2)*:
```solidity
File: src/cannon/PreimageOracle.sol

422:         if (msg.sender != tx.origin) revert NotEOA();

465:         if (msg.sender != tx.origin) revert NotEOA();

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

### <a name="L-5"></a>[L-5] Do not leave an implementation contract uninitialized
An uninitialized implementation contract can be taken over by an attacker, which may impact the proxy. To prevent the implementation contract from being used, it's advisable to invoke the `_disableInitializers` function in the constructor to automatically lock it when it is deployed. This should look similar to this:
```solidity
  /// @custom:oz-upgrades-unsafe-allow constructor
  constructor() {
      _disableInitializers();
  }
```

Sources:
- https://docs.openzeppelin.com/contracts/4.x/api/proxy#Initializable-_disableInitializers--
- https://twitter.com/0xCygaar/status/1621417995905167360?s=20

*Instances (2)*:
```solidity
File: src/dispute/DisputeGameFactory.sol

42:     constructor() OwnableUpgradeable() {

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

125:     constructor(

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="L-6"></a>[L-6] Division by zero not prevented
The divisions below take an input parameter which does not have any zero-value checks, which may lead to the functions reverting when zero is passed.

*Instances (3)*:
```solidity
File: src/cannon/MIPS.sol

429:                 state.lo = uint32(int32(_rs) / int32(_rt));

439:                 state.lo = _rs / _rt;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/MIPS.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

721:         uint256 a = highGasCharged / baseGasCharged;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="L-7"></a>[L-7] Duplicate import statements

*Instances (2)*:
```solidity
File: src/dispute/FaultDisputeGame.sol

14: import { Types } from "src/libraries/Types.sol";

17: import { Types } from "src/libraries/Types.sol";

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="L-8"></a>[L-8] External call recipient may consume all transaction gas
There is no limit specified on the amount of gas used, so the recipient can use up all of the transaction's gas, causing it to revert. Use `addr.call{gas: <amount>}("")` or [this](https://github.com/nomad-xyz/ExcessivelySafeCall) library instead.

*Instances (2)*:
```solidity
File: src/cannon/PreimageOracle.sol

792:         (bool success,) = _to.call{ value: bond }("");

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

759:         (bool success,) = _recipient.call{ value: recipientCredit }(hex"");

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="L-9"></a>[L-9] Initializers could be front-run
Initializers could be front-run, allowing an attacker to either set their own values, take ownership of the contract, and in the best case forcing a re-deployment

*Instances (4)*:
```solidity
File: src/dispute/DisputeGameFactory.sol

43:         initialize(address(0));

48:     function initialize(address _owner) public initializer {

49:         __Ownable_init();

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

157:     function initialize() public payable virtual {

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="L-10"></a>[L-10] Solidity version 0.8.20+ may not work on other chains due to `PUSH0`
The compiler for Solidity 0.8.20 switches the default target EVM version to [Shanghai](https://blog.soliditylang.org/2023/05/10/solidity-0.8.20-release-announcement/#important-note), which includes the new `PUSH0` op code. This op code may not yet be implemented on all L2s, so deployment on these chains will fail. To work around this issue, use an earlier [EVM](https://docs.soliditylang.org/en/v0.8.20/using-the-compiler.html?ref=zaryabs.com#setting-the-evm-version-to-target) [version](https://book.getfoundry.sh/reference/config/solidity-compiler#evm_version). While the project itself may or may not compile with 0.8.20, other projects with which it integrates, or which extend this project may, and those projects will have problems deploying these contracts/libraries.

*Instances (6)*:
```solidity
File: src/cannon/MIPS.sol

2: pragma solidity 0.8.15;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/MIPS.sol)

```solidity
File: src/cannon/PreimageKeyLib.sol

2: pragma solidity 0.8.15;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageKeyLib.sol)

```solidity
File: src/cannon/PreimageOracle.sol

2: pragma solidity 0.8.15;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

```solidity
File: src/cannon/libraries/CannonTypes.sol

2: pragma solidity 0.8.15;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/libraries/CannonTypes.sol)

```solidity
File: src/dispute/DisputeGameFactory.sol

2: pragma solidity 0.8.15;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

2: pragma solidity 0.8.15;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="L-11"></a>[L-11] Use `Ownable2Step.transferOwnership` instead of `Ownable.transferOwnership`
Use [Ownable2Step.transferOwnership](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable2Step.sol) which is safer. Use it as it is more secure due to 2-stage ownership transfer.

**Recommended Mitigation Steps**

Use <a href="https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/access/Ownable2Step.sol">Ownable2Step.sol</a>
  
  ```solidity
      function acceptOwnership() external {
          address sender = _msgSender();
          require(pendingOwner() == sender, "Ownable2Step: caller is not the new owner");
          _transferOwnership(sender);
      }
```

*Instances (2)*:
```solidity
File: src/dispute/DisputeGameFactory.sol

5: import { OwnableUpgradeable } from "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";

50:         _transferOwnership(_owner);

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol)

### <a name="L-12"></a>[L-12] Consider using OpenZeppelin's SafeCast library to prevent unexpected overflows when downcasting
Downcasting from `uint256`/`int256` in Solidity does not revert on overflow. This can result in undesired exploitation or bugs, since developers usually assume that overflows raise errors. [OpenZeppelin's SafeCast library](https://docs.openzeppelin.com/contracts/3.x/api/utils#SafeCast) restores this intuition by reverting the transaction when such an operation overflows. Using this library eliminates an entire class of bugs, so it's recommended to use it always. Some exceptions are acceptable like with the classic `uint256(uint160(address(variable)))`

*Instances (10)*:
```solidity
File: src/cannon/MIPS.sol

80:             return uint32(_dat & mask | (isSigned ? signed : 0));

234:                     state.preimageOffset += uint32(datLen);

235:                     v0 = uint32(datLen);

1021:                     uint32 mask = 0xFFFFFFFF ^ uint32(0xFF << (24 - (rs & 3) * 8));

1027:                     uint32 mask = 0xFFFFFFFF ^ uint32(0xFFFF << (16 - (rs & 2) * 8));

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/MIPS.sol)

```solidity
File: src/cannon/PreimageOracle.sol

539:             uint32(_input.length + metaData.bytesProcessed())

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

212:                 bond: uint128(msg.value),

397:                 parentIndex: uint32(_challengeIndex),

401:                 bond: uint128(msg.value),

628:         checkpoint.subgameIndex = uint32(finalCursor);

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)

### <a name="L-13"></a>[L-13] Upgradeable contract is missing a `__gap[50]` storage variable to allow for new storage variables in later versions
See [this](https://docs.openzeppelin.com/contracts/4.x/upgradeable#storage_gaps) link for a description of this storage variable. While some contracts may not currently be sub-classed, adding the variable now protects against forgetting to add it in the future.

*Instances (3)*:
```solidity
File: src/dispute/DisputeGameFactory.sol

5: import { OwnableUpgradeable } from "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";

19: contract DisputeGameFactory is OwnableUpgradeable, IDisputeGameFactory, ISemver {

42:     constructor() OwnableUpgradeable() {

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol)

### <a name="L-14"></a>[L-14] Upgradeable contract not initialized
Upgradeable contracts are initialized via an initializer function rather than by a constructor. Leaving such a contract uninitialized may lead to it being taken over by a malicious user

*Instances (12)*:
```solidity
File: src/cannon/PreimageOracle.sol

468:         if (metaData.claimedSize() == 0) revert NotInitialized();

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

```solidity
File: src/dispute/DisputeGameFactory.sol

5: import { OwnableUpgradeable } from "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";

19: contract DisputeGameFactory is OwnableUpgradeable, IDisputeGameFactory, ISemver {

42:     constructor() OwnableUpgradeable() {

43:         initialize(address(0));

48:     function initialize(address _owner) public initializer {

49:         __Ownable_init();

117:         proxy_.initialize{ value: msg.value }();

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol)

```solidity
File: src/dispute/FaultDisputeGame.sol

85:     bool internal initialized;

157:     function initialize() public payable virtual {

170:         if (initialized) revert AlreadyInitialized();

220:         initialized = true;

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)


## Medium Issues


| |Issue|Instances|
|-|:-|:-:|
| [M-1](#M-1) | `block.number` means different things on different L2s | 2 |
| [M-2](#M-2) | Centralization Risk for trusted owners | 2 |
| [M-3](#M-3) | Lack of EIP-712 compliance: using `keccak256()` directly on an array or struct variable | 4 |
### <a name="M-1"></a>[M-1] `block.number` means different things on different L2s
On Optimism, `block.number` is the L2 block number, but on Arbitrum, it's the L1 block number, and `ArbSys(address(100)).arbBlockNumber()` must be used. Furthermore, L2 block numbers often occur much more frequently than L1 block numbers (any may even occur on a per-transaction basis), so using block numbers for timing results in inconsistencies, especially when voting is involved across multiple chains. As of version 4.9, OpenZeppelin has [modified](https://blog.openzeppelin.com/introducing-openzeppelin-contracts-v4.9#governor) their governor code to use a clock rather than block numbers, to avoid these sorts of issues, but this still requires that the project [implement](https://docs.openzeppelin.com/contracts/4.x/governance#token_2) a [clock](https://eips.ethereum.org/EIPS/eip-6372) for each L2.

*Instances (2)*:
```solidity
File: src/cannon/PreimageOracle.sol

554:         proposalBlocks[msg.sender][_uuid].push(uint64(block.number));

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

```solidity
File: src/dispute/DisputeGameFactory.sol

103:         bytes32 parentHash = blockhash(block.number - 1);

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol)

### <a name="M-2"></a>[M-2] Centralization Risk for trusted owners

#### Impact:
Contracts have owners with privileged rights to perform admin tasks and need to be trusted to not perform malicious updates or drain funds.

*Instances (2)*:
```solidity
File: src/dispute/DisputeGameFactory.sol

199:     function setImplementation(GameType _gameType, IDisputeGame _impl) external onlyOwner {

205:     function setInitBond(GameType _gameType, uint256 _initBond) external onlyOwner {

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol)

### <a name="M-3"></a>[M-3] Lack of EIP-712 compliance: using `keccak256()` directly on an array or struct variable
Directly using the actual variable instead of encoding the array values goes against the EIP-712 specification https://github.com/ethereum/EIPs/blob/master/EIPS/eip-712.md#definition-of-encodedata. 
**Note**: OpenSea's [Seaport's example with offerHashes and considerationHashes](https://github.com/ProjectOpenSea/seaport/blob/a62c2f8f484784735025d7b03ccb37865bc39e5a/reference/lib/ReferenceGettersAndDerivers.sol#L130-L131) can be used as a reference to understand how array of structs should be encoded.

*Instances (4)*:
```solidity
File: src/cannon/PreimageOracle.sol

589:         if (keccak256(abi.encode(_stateMatrix)) != _preState.stateCommitment) revert InvalidPreimage();

599:         if (keccak256(abi.encode(_stateMatrix)) == _postState.stateCommitment) revert PostStateMatches();

630:         if (keccak256(abi.encode(stateMatrix)) == _postState.stateCommitment) revert PostStateMatches();

669:         if (keccak256(abi.encode(_stateMatrix)) != _preState.stateCommitment) revert InvalidPreimage();

```
[Link to code](https://github.com/code-423n4/2024-07-optimism/tree/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)

