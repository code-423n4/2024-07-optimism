# Optimism Superchain audit details
- Total Prize Pool: 200,000 $OP + $80,000 in USDC
  - Open Competition Pool: 200,000 $OP
    - HM awards: 194,700 $OP*
    - QA awards: 5,000 $OP
    - Scout awards: 300 $OP
  - Pro League Side Pool: $80,000 in USDC
- Join [C4 Discord](https://discord.gg/code4rena) to register
- Submit findings [using the C4 form](https://code4rena.com/contests/2024-07-optimism-superchain/submit)
- [Read our guidelines for more details](https://docs.code4rena.com/roles/wardens)
- Starts July 15, 2024 20:00 UTC
- Ends July 29, 2024 20:00 UTC

***❗️Breakdown for HM awards:**
- 5% Hunter bonus (10,000 $OP)
- 5% Gatherer bonus (10,000 $OP)
- 10% Dark Horse bonus (20,000 $OP)
- HM awards (154,700 $OP)

**Dark Horse bonus:**
- Everyone competes against Pro League teams in order to win a Dark Horse bonus (ranking by [HM award algorithm](https://docs.code4rena.com/awarding/incentive-model-and-awards#high-and-medium-risk-bugs)).
- The Dark Horse bonus is distributed from the HM pool.
- 10% of the total awards (20,000) are split amongst everyone who beats or ties one Pro League team. (If no one beats or ties a Pro League team, no Dark Horse bonus is awarded.)

**Pro League side pool:**
- Pro League side pool is split 60/40 among Pro League teams based on their [Gatherer score](https://docs.code4rena.com/awarding/incentive-model-and-awards#bonuses-for-top-competitors).
- Pro League teams also compete for a portion of HM awards but are not eligible for Hunter/Gatherer/Dark Horse bonuses.

_Note: The judge/validator is performing their role for $0 in order to maximize the awards pool._

**IMPORTANT NOTE:** Prior to receiving payment from this audit you MUST complete a basic KYC (Know Your Customer) process.

You do not have to be KYC'd before submitting bugs. But you must a) successfully complete the basic KYC process within 30 days of the audit end date in order to receive awards. This applies to all audit participants, including wardens, teams, judges, validators, and scouts.

## Automated Findings / Publicly Known Issues
All issues from the [Spearbit Coinbase audit](https://github.com/spearbit/portfolio/blob/master/pdfs/Spearbit-Coinbase-Security-Review-June-2024.pdf) on these contracts is considered known, including:

1. `loadPrecompilePreimagePart()` can be called with too little gas so that the precompile reverts and the result overwrites valid data, which can be used to prove an invalid execution trace.

2. If the contract is configured so that `SPLIT_DEPTH + 1 = MAX_DEPTH` and the last game step is a defend action, ancestor lookup can misbehave and lead to an out of bounds array access.

3. If an attacker has more funds than the honest defender, they can continually open more subgames until the defender runs out of money and the challenger takes all their bonds (aka [proof of whale](https://ethresear.ch/t/fraud-proofs-are-broken/19234)).

4. If a defender is going to lose the game, they can call `challengeRootL2Block()` on themselves, and they will get priority and take their own top level bond payout, instead of paying them to the real challenger.

5. The Go program cannot garbage collect on the MIPS VM, with garbage collection symbols explicitly patched out. Theoretically this could allow for overflowing the heap.

6. The 4naly3er report can be found [here](https://github.com/code-423n4/2024-07-optimism/blob/main/4naly3er-report.md).

_Note for C4 wardens: Anything included in this `Automated Findings / Publicly Known Issues` section is considered a publicly known issue and is ineligible for awards._

# Overview

Welcome to the Fault Dispute Game competitive audit!

Optimism [deployed its Fault Dispute Game on Mainnet last month](https://vote.optimism.io/proposals/72085170435228531173144599119267762084652443676555508407874836206178427511368), with many carefully audited safeguards surrounding the game to protect the protocol in the event of a exploit. As they move towards confidence in removing those safeguards, one of the key pieces is a competitive audit with the ecosystem's top talent.

This is where you come in.

## What Is A Fault Dispute Game?

At a high level, the Fault Dispute Game is a contract that allows anyone to post an ETH bond along with a claim about an output root that represents the state of L2.

If nobody challenges the claim for a period of time, it is accepted, and the root is considered valid. This means that users can withdraw ETH through the bridge based on the state represented by that root.

If someone does challenge the claim, the game enters a dispute phase. The challenger and the defender play a "bisection game" to determine which L2 block the disagree on (continually adding more bonds as they go), and then play a further game to get down to the individual trace instruction in the block executed in a MIPS VM that caused the disagreement. From there, the Solidity contract can execute that instruction to determine who was right.

The winner gets the bonds and, if the winner is the defender, their claim is considered valid.

## What Are The Current Safeguards?

Currently, there are two safeguards in place around this game:

1) After a game is settled, there is a time period during which withdrawals against that output root are not allowed. This gives the Optimism team time to respond and reject the claim if an exploit has occurred in the game.

2) Similarly, after a game is settled, all the ETH that should be paid out is held in a DelayedWETH contract. It can similarly be reassigned by the Optimism team in the event of an exploit.

## What Is The Goal of This Audit?

For the sake of this audit, you should pretend the safeguards don't exist.

We aren't looking for ways to get around the safeguards, or bugs in the Optimism Portal (although those would both be eligible through Optimism's [bug bounty program](https://immunefi.com/bug-bounty/optimism)).

Instead, we are looking for any flaws in the game and MIPS VM logic.

In other words:
- _Can a dishonest actor ever prove a false claim?_
- _Can an honest actor ever fail to prove a true claim?_
- _Does the game misbehave in any other ways that could cause problems if there were no safeguards?_

## How Does Optimism Work?

To understand the context for how Optimism works (and where this Fault Dispute Game fits in), please see the following resources:

1) [Norswap's Bedrock Whitepaper](https://norswap.notion.site/Bedrock-Whitepaper-020f1a8e80ee4045ac8a84d22736b714): While this is slightly outdated (it's from before the Fault Dispute Game and before EIP4844 enabled blobs), it's the most understandable overview of the full Optimism protocol from end to end.

2) [Optimism Specs](https://specs.optimism.io/root.html): These specs are the most precise definition of how each part of the Optimism protocol workss. The [Fault Proof](https://specs.optimism.io/fault-proof/index.html) section of the specs will be particularly relevant for this audit.

3) Code Walkthrough (11:30am ET on 7/15): A video walkthrough of the Fault Dispute Game code by Clabby (one of the core developers of the fault game), explaining how the contracts were and how they fit into the larger system.

## A Note on POCs

Because many issues submitted in this audit will be about complex interactions between the Attacker and Challenger, it is important that POCs accurately demonstrate this logic.

To that end, please use the [E2E Cannon Tests](https://github.com/ethereum-optimism/optimism/blob/develop/op-e2e/faultproofs/output_cannon_test.go) as a starting point for all POCs that involve interactions between various actors. See [Running tests](#running-tests) for more detailed instructions.

Forge tests are acceptable for any POCs that only need to demonstrate small or isolated properties, but the smart contract test suite is not configured with the full honest challenger behavior nor does it use the actual fault proof VM as the step function. POCs that try to prove complex behavior with Forge tests will not be accepted.

## Links

- **Previous audits:**  There was a [Sherlock contest](https://github.com/sherlock-audit/2024-02-optimism-2024-judging/) on the safeguards around the Fault Dispute Game. Some issues in the game itself were submitted, despite being out of scope. There was also a [Spearbit Coinbase audit](https://github.com/spearbit/portfolio/blob/master/pdfs/Spearbit-Coinbase-Security-Review-June-2024.pdf).
- **Documentation:** https://specs.optimism.io/fault-proof/stage-one/fault-dispute-game.html
- **Website:** https://optimism.io/
- **X/Twitter:** https://twitter.com/Optimism
- **Discord:** https://discord.com/invite/optimism

---

# Scope


### Files in scope


| **Contract**                         | **SLOC** |
|--------------------------------------|----------|
| [src/dispute/FaultDisputeGame.sol](https://github.com/code-423n4/2024-07-optimism/blob/main/packages/contracts-bedrock/src/dispute/FaultDisputeGame.sol)     | 469      |
| [src/dispute/DisputeGameFactory.sol](https://github.com/code-423n4/2024-07-optimism/blob/main/packages/contracts-bedrock/src/dispute/DisputeGameFactory.sol)   | 125      |
| [src/cannon/MIPS.sol](https://github.com/code-423n4/2024-07-optimism/blob/main/packages/contracts-bedrock/src/cannon/MIPS.sol)                                 | 690      |
| [src/cannon/PreimageOracle.sol](https://github.com/code-423n4/2024-07-optimism/blob/main/packages/contracts-bedrock/src/cannon/PreimageOracle.sol)              | 470      |
| [src/cannon/PreimageKeyLib.sol](https://github.com/code-423n4/2024-07-optimism/blob/main/packages/contracts-bedrock/src/cannon/PreimageKeyLib.sol)             | 26       |
| [src/cannon/libraries/CannonTypes.sol](https://github.com/code-423n4/2024-07-optimism/blob/main/packages/contracts-bedrock/src/cannon/libraries/CannonTypes.sol) | 67       |
| **TOTAL**                            | **1847** |


Note the current version of the contracts are deployed on Mainnet to:

| Contract                            | Address                                    |
|-------------------------------------|--------------------------------------------|
| DisputeGameFactory (proxy)          | [0xe5965Ab5962eDc7477C8520243A95517CD252fA9](https://etherscan.io/address/0xe5965Ab5962eDc7477C8520243A95517CD252fA9) |
| DisputeGameFactory (implementation) | [0xc641a33cab81c559f2bd4b21ea34c290e2440c2b](https://etherscan.io/address/0xc641a33cab81c559f2bd4b21ea34c290e2440c2b) |
| FaultDisputeGame (implementation)        | [0x4146DF64D83acB0DcB0c1a4884a16f090165e122](https://etherscan.io/address/0x4146DF64D83acB0DcB0c1a4884a16f090165e122) |
| MIPS                                | [0x0f8EdFbDdD3c0256A80AD8C0F2560B1807873C9c](https://etherscan.io/address/0x0f8EdFbDdD3c0256A80AD8C0F2560B1807873C9c) |
| PreimageOracle                      | [0xD326E10B8186e90F4E2adc5c13a2d0C137ee8b34](https://etherscan.io/address/0xD326E10B8186e90F4E2adc5c13a2d0C137ee8b34) |

(All other Optimism contract addresses can be found here: https://docs.optimism.io/chain/addresses)

### Files out of scope

Any file not listed in the table above is out of scope for this audit.

While the rest of the repo may be useful to look at for context, and for implications of how bugs in the Fault Dispute Game might be exploited, this audit is focused only on issues from the in scope contracts themselves.

If other issues are found, Optimism has an active [bug bounty program](https://immunefi.com/bug-bounty/optimism) where they can be submitted.

## Scoping Q &amp; A

### General questions


| Question                                | Answer                       |
| --------------------------------------- | ---------------------------- |
| ERC20 used by the protocol              |       None             |
| Test coverage                           | -                          |
| ERC721 used  by the protocol            |            None              |
| ERC777 used by the protocol             |           None                |
| ERC1155 used by the protocol            |           None            |
| Chains the protocol will be deployed on | Ethereum            |


### External integrations (e.g., Uniswap) behavior in scope:


| Question                                                  | Answer |
| --------------------------------------------------------- | ------ |
| Enabling/disabling fees (e.g. Blur disables/enables fees) | No   |
| Pausability (e.g. Uniswap pool gets paused)               |  No   |
| Upgradeability (e.g. Uniswap gets upgraded)               |   No  |


### EIP compliance checklist
N/A


# Additional context

## Main invariants

1) Honest actors should always be able to win the Fault Dispute Game.
2) Dishonest actors should never be able to win the Fault Dispute Game.


## Attack ideas (where to focus for bugs)
1) The Fault Dispute Game logic is complex. Is there any way to get it to misbehave such that an invalid claim is accepted, or a valid claim is rejected?
2) The Fault Dispute Game relies heavily on the game clock. Is there a way to trick the game clock such that an honest actor can be forced to time out?
3) The on chain MIPS processor must reflect the execution trace created by the offchain op-program. Is it always accurate?
4) The Preimage Oracle is trusted to only provide accurate data. Is there a way to get invalid data added and use it to prove an invalid execution trace?


## All trusted roles in the protocol

There are trusted roles in the protocol (for example, enforcing the safeguards on the in scope contracts), but they aren't relevant for the scope of this audit, which focuses exclusively on whether the game can be made to resolve incorrectly.


## Describe any novel or unique curve logic or mathematical models implemented in the contracts:

N/A


## Running tests

Dependencies:
1. Golang compiler (v1.21.1)
2. Rust compiler (>= 1.75.0)
3. pnpm (any version)
4. foundry suite (version = 63fff3510408b552f11efb8196f48cfe6c1da664)

```bash
git clone https://github.com/code-423n4/2024-07-optimism
cd 2024-07-optimism/packages/contracts-bedrock
```

To run Foundry tests:
```
forge install
pnpm build:go-ffi
# running tests for the entire bedrock contracts would take a while, the following command
# would filter only to the dispute tests
forge test --mp "*dispute*"
```

To run e2e tests:
```
# from the root of the project
make install-geth
make cannon-prestate
make devnet-allocs

# from the op-e2e folder
make test-cannon
```

If tests are failing for an unknown reason:
- ensure you have the latest version of foundry installed: `pnpm update:foundry` (requires `jq` to be installed)
- try deleting the `packages/contracts-bedrock/forge-artifacts` directory
- if the above step doesn't fix the error, try `pnpm clean`

## Miscellaneous
Employees of Optimism and employees' family members are ineligible to participate in this audit.
