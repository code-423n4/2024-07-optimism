# Optimism Superchain audit details
- Total Prize Pool: 200,000 OP + $80,000 in USDC 
  - Main Warden Pool: 200,000 OP
    - HM awards: XXXX OP - *
    - QA awards: 5,000 OP
    - Judge awards: XXXX OP
    - Validator awards: XXXX OP
    - Scout awards: $500 USDC
  - Pro League Side Pool: $80,000 in USDC
    - HM awards: $78,000 in USDC
    - QA awards: $2,000 in USDC
- Join [C4 Discord](https://discord.gg/code4rena) to register
- Submit findings [using the C4 form](https://code4rena.com/contests/2024-07-optimism-superchain/submit)
- [Read our guidelines for more details](https://docs.code4rena.com/roles/wardens)
- Starts July 15, 2024 20:00 UTC
- Ends July 29, 2024 20:00 UTC

*❗️Awarding Rules: the total HM awards for the Main Pool will be broken down as follows:
- 5% Hunter bonus (5,000 OP)
- 5% Gatherer bonus (5,000 OP)
- 5-10% Dark Horse bonus (5,000 - 10,000 OP)
- 80-85% HM awards (XXXX - XXXX OP)

Pro League teams are also competing for a portion of the HM awards, but will not receive the Hunter/Gather/Dark Horse bonuses.  More details here: [Dark Horse Era of C4](link to a page with Dark horse info).

## Automated Findings / Publicly Known Issues

The 4naly3er report can be found [here](https://github.com/code-423n4/2024-07-optimism/blob/main/4naly3er-report.md).

_Note for C4 wardens: Anything included in this `Automated Findings / Publicly Known Issues` section is considered a publicly known issue and is ineligible for awards._

# Overview

[ ⭐️ SPONSORS: add info here ]

## Links

- **Previous audits:**  N/A
- **Documentation:** https://specs.optimism.io/fault-proof/stage-one/fault-dispute-game.html
- **Website:** https://optimism.io/
- **X/Twitter:** https://twitter.com/Optimism
- **Discord:** https://discord.com/invite/optimism

---

# Scope


### Files in scope


| **Contract**                         | **SLOC** |
|--------------------------------------|----------|
| src/dispute/FaultDisputeGame.sol     | 469      |
| src/dispute/DisputeGameFactory.sol   | 125      |
| src/cannon/MIPS.sol                  | 690      |
| src/cannon/PreimageOracle.sol        | 470      |
| src/cannon/PreimageKeyLib.sol        | 26       |
| src/cannon/libraries/CannonTypes.sol | 67       |
| **TOTAL**                            | **1847** |


Note the current version of the contracts are deployed to the following:

| Contract                            | Address                                    |
|-------------------------------------|--------------------------------------------|
| DisputeGameFactory (proxy)          | [0xe5965Ab5962eDc7477C8520243A95517CD252fA9](https://etherscan.io/address/0xe5965Ab5962eDc7477C8520243A95517CD252fA9) |
| DisputeGameFactory (implementation) | [0xc641a33cab81c559f2bd4b21ea34c290e2440c2b](https://etherscan.io/address/0xc641a33cab81c559f2bd4b21ea34c290e2440c2b) |
| DisputeGame (implementation)        | [0x4146DF64D83acB0DcB0c1a4884a16f090165e122](https://etherscan.io/address/0x4146DF64D83acB0DcB0c1a4884a16f090165e122) |
| MIPS                                | [0x0f8EdFbDdD3c0256A80AD8C0F2560B1807873C9c](https://etherscan.io/address/0x0f8EdFbDdD3c0256A80AD8C0F2560B1807873C9c) |
| PreimageOracle                      | [0xD326E10B8186e90F4E2adc5c13a2d0C137ee8b34](https://etherscan.io/address/0xD326E10B8186e90F4E2adc5c13a2d0C137ee8b34) |

### Files out of scope
Any file not listed in the table above

## Scoping Q &amp; A

### General questions


| Question                                | Answer                       |
| --------------------------------------- | ---------------------------- |
| ERC20 used by the protocol              |       None             |
| Test coverage                           | ✅ SCOUTS: Please populate this after running the test coverage command                          |
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



Run `pnpm clean` and rerun the tests if tests are failing for an unknown reason.


```bash
git clone https://github.com/code-423n4/2024-07-optimism
cd 2024-07-optimism/packages/contracts-bedrock
git submodule update --init --recursive

npm i
forge test
```

To run code coverage
```bash
forge coverage
```


## Miscellaneous
Employees of Optimism and employees' family members are ineligible to participate in this audit.
