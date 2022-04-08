# GoodGhosting Hodl Pool

GoodGhosting is a DeFi Protocol currently live on Polygon & Celo, which is revolutionizing the savings domain by gamifying it and rewarding regular savers.

You can read more about the [V0 Version](https://github.com/Good-Ghosting/goodghosting-protocol-v0#readme) to get a better understanding of the protocol.

With the GoodGhosting Hodl Pool, we aim to improve on the protocol and make it more fair, rewarding and more yield generating opportunities for regular savers in addition to the hodl feature.

## Notable Features in Hodl Pool

- **Different segment and waiting round length** as the word "hodl" the hodl pool aims to reduce the player's interaction with smart contract so the hodl pools can have just 1 deposit segment and 3/6 month waiting period.

- **Flexible Amount Deposit Pools** yes you heard it correct, this option will enabled while deployment of pool contract, there will be a max. amount limit but players can choose any valid amount but once you choose the amount you have to deposit the same amount in each segment.

- **Accounting of the interest and rewards distributed to winners** To introduce fairness unlike v0, the share of interest and rewards for winners will be decided by how much you deposit (in case of a flexible deposit pool) and how early you deposit in each segment.

- **Multiple Yield Strategies** the new hodl pool smart contract architecture allows to have multiple sources for yield strategies other than aave and moola like curve, mobius etc and many more in the future.

## Types of Pools
- **Fixed Deposit Pool with same waiting segment duration** Native Pool where the deposit amount is fixed for each player and the waiting round duration is fixed too.

- **Fixed Deposit Pool with waiting round duration more than payment segment duration** Fixed Deposit Hodl Pool basically which will have less player interaction in terms of sending transactions i.e a pool with 1 deposit segment of 1 week and 3 months of waiting round.

- **Flexible Deposit Pool with same waiting segment duration** Flexible Deposit Amount Pool where the deposit amount is decided by the player while joining and the waiting round duration is fixed too.

- **Flexible Deposit Pool with waiting round duration more than payment segment duration** Flexible Deposit Amount Hodl Pool basically which will have less player interaction in terms of sending transactions i.e a pool with 1 deposit segment of 1 week and 3 months of waiting round.

- **Transactional Token Pool** Fixed or Flexible Deposit pool with transactional token as deposit for example a pool with matic as deposit token.

- **Deposit Token same as reward Token** Fixed or Flexible Deposit pool with deposit token and reward token same for example a pool with wmatic as deposit token.

## Pool Interest Accounting Math
As mentioned above with this new version of the GoodGhosting Protocol we aim to introduce fairness in terms of interest and reward generation for winners, and we are doing this by computing player share % which is computed based on the deposit amount made by each player and how early does a player pay in each segment, so below is an explanation of the math behind this feature.

In the new version of the [Pool Smart Contract](https://github.com/Good-Ghosting/goodghosting-protocol-v1/blob/master/contracts/Pool.sol) we have a couple of mappings defined ```playerIndex[player_address][segment_number] & cummalativePlayerIndexSum[segment_number]```

```
Now as each player starts making deposits the mappings are updated as follows:
playerIndex = deposit_amount * (MULTIPLIER_CONSTANT) / block.timestamp

for (each segment paid by the player) {
cummalativePlayerIndexSum[current_segment] += playerIndex
}

cummalativePlayerIndexSum as you see is updated for every new deposit for each segment (reason for updating it in each segment is explained in the next section)

here MULTIPLIER_CONSTANT is used since solidity cannot handle decimal values it is a very large constant 10**8

so the playerIndex updates for each player during each deposit.
```

So once the game get's over when the players withdraw the interest & reward share is calculated as:

```
uint playerIndexSum = 0;
for (each segment paid by the player) {
playerIndexSum += playerIndex
}

playerSharePercentage = playerIndexSum * 100 / cummalativePlayerIndexSum[last_segment]

playerSharePercentage is the % of funds the winners get from the total game interest and total rewards generated by the game.
```
Couple of examples
- A Game with 2 winners who deposited different amounts and at different times
```
There are 2 players in the game with only 1 segment and it's a flexible deposit pool, player1 deposits 10 DAI & player 2 deposits 100 DAI, but player 1 deposits early than player 2.

player1Index = 10 / 5 = 2
player2Index = 100 / 20 = 5

so hence cummalativePlayerIndexSum will be 7 and even though player 2 deposits late but the amount is 10x more than player 1 so player2 in the end get's about 72 % of the rewards and player1 get's the remaining 28 %.
```

- A Game with 2 players, 1 early withdrawal
```
There are 2 players in the game with only 1 segment and it's a flexible deposit pool, player1 deposits 10 DAI & player 2 deposits 100 DAI, but player 1 deposits early than player 2 

player1Index = 10 / 5 = 2
player2Index = 100 / 20 = 5

so hence cummalativePlayerIndexSum will be 7, now the twist is that player 2 early withdraw's so cummalativePlayerIndexSum then becomes 2, and hence player 1 get's all the interest and rewards earned.
```



## Emergency Scenario
With Hodl Pools especially there comes a risk of funds being locked in the external protocol in case something happens or if an external protocol utilized by one of the goodghosting pools migrates to a new contract in the middle of a game. So to handle these scenario's we have a introduced a new function in the smart contract (enableEmergencyWithdraw)[https://github.com/Good-Ghosting/goodghosting-protocol-v1/blob/master/contracts/Pool.sol#L572] which can only be called by the contract deployer aka the admin.


Once this function is called, it updates the last segment value to current segment & makes the emergency flag true in the smart contract, players then who have deposited in the prev. segment i.e current segment - 1 are all considered as winners and they can withdraw their funds immeditately once the emergency flag is enabled.

**NOTE** - Handling this emergency early exit is the reason `cummalativePlayerIndexSum` is a mapping.



# Smart Contract Overview

<img width="1368" alt="Screenshot 2022-03-10 at 9 34 58 AM" src="https://user-images.githubusercontent.com/26670962/157606809-2df36e2f-9c71-4bed-b291-d37377095ef2.png">

> As you can see in order to make our contracts modular we have divided the contracts into two types, the pool contract which holds all the core game/pool logic and the yield strategy contracts so in a way the pool contract is the parent contract and players cannot directly interact with the strategy contracts, only the pool contract can, so while deploying the pool we set the address of the strategy we want to use for that pool, below is an in depth explanation of each contract.

<br/>

* **[Pool](https://github.com/Good-Ghosting/goodghosting-protocol-v1/blob/master/contracts/Pool.sol)** is the core contract with only the game logic in it, it's the parent contract through which players are able to make deposits into various yield strategy contracts and withdraw funds.

* **[IStrategy](https://github.com/Good-Ghosting/goodghosting-protocol-v1/blob/master/contracts/strategies/IStrategy.sol)** is the interface that all strategy contracts inherit so that it becomes straightforward to plug and play any strategy in the pool contract.

* **[AaveStrategyV3](https://github.com/Good-Ghosting/goodghosting-protocol-v1/blob/master/contracts/strategies/AaveStrategyV3.sol)** is responsible for depositing funds that it gets from the pool contract to aave v3 and withdraw the funds from there and send back to the pool contract.

* **[AaveStrategy](https://github.com/Good-Ghosting/goodghosting-protocol-v1/blob/master/contracts/strategies/AaveStrategy.sol)** is responsible for depositing funds that it gets from the pool contract to aave v2/moola and withdraw the funds from aave/moola and send back to the pool contract.

* **[CurveStrategy](https://github.com/Good-Ghosting/goodghosting-protocol-v1/blob/master/contracts/strategies/CurveStrategy.sol)** is responsible for depositing funds that it gets from the pool contract to curve stable/volatile pools and withdraw the funds from curve stable/volatile pools and send back to the pool contract, current pools supported are AAVE Stable Pool `0x445FE580eF8d70FF569aB36e80c647af338db351` & Atricrypto Volatile Pool `0x1d8b86e3d88cdb2d34688e87e72f388cb541b7c8`.

* **[MobiusStrategy](https://github.com/Good-Ghosting/goodghosting-protocol-v1/blob/master/contracts/strategies/MobiusStrategy.sol)**: is responsible for depositing funds that it gets from the pool contract to any mobius liquidity pool and withdraw the funds from that mobius liquidity pool and send back to the pool contract, current pools that were tested with are cUSD / DAI Pool `0xF3f65dFe0c8c8f2986da0FEc159ABE6fd4E700B4` & cUSD / USDC Pool `0x9906589Ea8fd27504974b7e8201DF5bBdE986b03`

# Development
The repository uses both hardhat and truffle, hardhat is used for aave strategy based pool deployments and unit tests and for other strategy fork tests and deployments truffle is used.

## Setup

Install Truffle.
```bash
yarn add global truffle
```

Install Ganache for having a local dev Ethereum network.
```bash
yarn add global ganache ganache-cli
```

Create a local `.env` file by copying the sample `.env.sample` file available in the root folder (`cp .env.sample .env`). After your `.env` file is created, edit it with appropriate values for the variables.

Install Project dependencies
```bash
yarn install
```

## Common Development Commands

Compile contracts
```bash
yarn compile
```


# Tests

## Unit Tests


**Requirement:** Make sure the `FORKING` var is set false before running the unit test suite.

To run the unit tests use either
`yarn test`

To run test coverage: `yarn coverage`

NOTE - If you run any test command after `yarn coverage` you will see an error 

```
An unexpected error occurred:

test/fork/pool.aave.emergency.withdraw.test.ts:5:34 - error TS2307: Cannot find module '../../artifacts/contracts/aave/ILendingPoolAddressesProvider.sol/ILendingPoolAddressesProvider.json' or its corresponding type declarations.

5 import * as lendingProvider from "../../artifacts/contracts/aave/ILendingPoolAddressesProvider.sol/ILendingPoolAddressesProvider.json";
                                   ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
test/fork/pool.aave.emergency.withdraw.test.ts:6:38 - error TS2307: Cannot find module '../../artifacts/contracts/aave/IncentiveController.sol/IncentiveController.json' or its corresponding type declarations.

6 import * as incentiveController from "../../artifacts/contracts/aave/IncentiveController.sol/IncentiveController.json";
                                       ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
test/fork/pool.aave.emergency.withdraw.test.ts:7:25 - error TS2307: Cannot find module '../../artifacts/contracts/mock/MintableERC20.sol/MintableERC20.json' or its corresponding type declarations.

7 import * as wmatic from "../../artifacts/contracts/mock/MintableERC20.sol/MintableERC20.json";
                          ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
test/fork/pool.aave.emergency.withdraw.test.ts:8:31 - error TS2307: Cannot find module '../../artifacts/contracts/mock/LendingPoolAddressesProviderMock.sol/LendingPoolAddressesProviderMock.json' or its corresponding type declarations.

8 import * as dataProvider from "../../artifacts/contracts/mock/LendingPoolAddressesProviderMock.sol/LendingPoolAddressesProviderMock.json";
```

**just ignore this error and run the command again**

## Integration Tests Using Forked Networks
### Setup
To run the integrated test scenarios forking from Mainnet (Polygon or Celo) you'll have to:
- Configure `WHALE_ADDRESS_FORKED_NETWORK` in your `.env` file, as you see the .env.example file the whale address is `0x075e72a5edf65f0a5f44699c7654c1a76941ddc8` for polygon & `0x5776b4893faca32A9224F18950406c9599f3B013` for celo.

- Review the deployment configs ([deploy-config.js file](./deploy-config.js)) prior to executing the test on the forked network.

### Steps
#### Polygon
- **Aave V2/V3 Strategy Based Pool** As mentioned above we use hardhat for this, after doing the setup mentioned above, the next step is to set the `FORKING` var in your .env file as `true`, next in your [config](https://github.com/Good-Ghosting/goodghosting-protocol-v1/blob/master/hardhat.config.ts#L62) you set your desired rpc url, currently a public rpc is set. Then you just run `yarn test`.

- **Curve Strategy Based Pool** As mentioned above we use truffle for this, so open a new terminal window and run  `ganache-cli -f <Your Polygon RPC> -m "clutchaptain shoe salt awake harvest setup primary inmate ugly among become" -i 999 --unlock {WHALE_ADDRESS_FORKED_NETWORK}` and in the second window run `yarn test:fork:polygon:curve` for fixed deposit pool & `yarn test:fork:variable:polygon:curve` for variable deposit pool.

#### Celo
Since hardhat currently does not support celo, so we use truffle for celo fork tests. To start open another terminal window and run
`ganache-cli -f https://forno.celo.org/ -m "clutchaptain shoe salt awake harvest setup primary inmate ugly among become" -i 999 --unlock {WHALE_ADDRESS_FORKED_NETWORK}` and in the second window run 
`yarn test:fork:celo:mobius` for fixed deposit mobius strategy pool, `yarn test:fork:celo:moola` for fixed deposit moola strategy pool, `yarn test:fork:variable:celo:mobius` for variable deposit mobius strategy pool & `yarn test:fork:variable:celo:moola` for variable deposit moola strategy pool.

# Security Tools
There's a few automated security tools that could be integrated with the development process. Currently, we use [Slither](https://github.com/crytic/slither) to help identify well-known issues via static analysis. Other tools may be added in the near future as part of the continuous improvement process.

## Slither
Make sure you install Slither by following the instructions available on [Slither's](https://github.com/crytic/slither) github page. Note: it requires Python, so you may need to install it before you're able to use Slither.

Slither can be executed with the following command:

```bash
slither .
```


# Contract Deployment
## Polygon
- **Aave V3 Strategy Based Pool** Start by setting the `MNEMONIC` var (which is the 12 word seed phrase in your wallet) & the `RPC` var in the .env file & then make sure you have the [right deployment configs set](https://github.com/Good-Ghosting/goodghosting-protocol-v1/blob/master/deploy.config.ts)(if a **whitelisted pool** needs to be deployed make sure the merkle root is set and the [isWhitelisted var](https://github.com/Good-Ghosting/goodghosting-protocol-v1/blob/master/deploy.config.ts#L135) is true), then just run `yarn deploy:polygon-aaveV3`.

You will see something like this:

```
Starting migrations...
======================
> Network name:    'polygon-aaveV3'
> Network id:      137
> Block gas limit: 30000000 (0x1c9c380)


2_deploy_contracts.js
=====================

   Deploying 'AaveStrategyV3'
   ------------------------
   > transaction hash:    0x3d0445201814629cf0eea2e68f0c034a288708ec62e5c50ef558f3fdff30b873
   > Blocks: 2            Seconds: 9
   > contract address:    0x7f8bA69d2D7bD4490AB0aa35B92e29B845aaB7fA
   > block number:        26186433
   > block timestamp:     1647862592
   > account:             0xf88b0247e611eE5af8Cf98f5303769Cba8e7177C
   > balance:             12.653319168891911123
   > gas used:            2004678 (0x1e96c6)
   > gas price:           32 gwei
   > value sent:          0 ETH
   > total cost:          0.064149696 ETH

   Pausing for 2 confirmations...
   ------------------------------
   > confirmation number: 3 (block: 26186439)

   Replacing 'SafeMath'
   --------------------
   > transaction hash:    0x651ba018aa6709ef01644b45b353111814630922d4ae03d976a7de4e8e37adc3
   > Blocks: 3            Seconds: 5
   > contract address:    0xAE130829ffeD8249BE3323289f15E4Bfd0770203
   > block number:        26186444
   > block timestamp:     1647862614
   > account:             0xf88b0247e611eE5af8Cf98f5303769Cba8e7177C
   > balance:             12.651008224891911123
   > gas used:            72217 (0x11a19)
   > gas price:           32 gwei
   > value sent:          0 ETH
   > total cost:          0.002310944 ETH

   Pausing for 2 confirmations...
   ------------------------------
   > confirmation number: 2 (block: 26186449)

   Replacing 'Pool'
   ----------------
   > transaction hash:    0xb16d515ed33d945d1c38be20c384d314ab0e129a602fc08eb44b4963f6bfcca1
   > Blocks: 5            Seconds: 10
   > contract address:    0x43a84D3BC0Fb6CFC93c7F9D08d8Be46a500bd9f3
   > block number:        26186454
   > block timestamp:     1647862634
   > account:             0xf88b0247e611eE5af8Cf98f5303769Cba8e7177C
   > balance:             12.504153056891911123
   > gas used:            4589224 (0x4606a8)
   > gas price:           32 gwei
   > value sent:          0 ETH
   > total cost:          0.146855168 ETH

   Pausing for 2 confirmations...
   ------------------------------
   > confirmation number: 2 (block: 26186461)



----------------------------------------------------
GoodGhosting Holding Pool deployed with the following arguments:
----------------------------------------------------

Network Name: polygon-aaveV3
Contract's Owner: 0xf88b0247e611eE5af8Cf98f5303769Cba8e7177C
Inbound Currency: 0x8f3Cf7ad23Cd3CaDbD9735AFf958023239c6A063
Maximum Flexible Segment Payment Amount: 0
Segment Count: 3
Segment Length: 604800 seconds
Waiting Segment Length: 604800 seconds
Segment Payment: 3 dai (3000000000000000000 wei)
Early Withdrawal Fee: 1%
Custom Pool Fee: 1%
Max Quantity of Players: 115792089237316195423570985008687907853269984665640564039457584007913129639935
Flexible Deposit Pool: false
Transactional Token Depsoit Pool: false
Strategy: 0x7f8bA69d2D7bD4490AB0aa35B92e29B845aaB7fA
Lending Pool Provider: 0xa97684ead0e402dC232d5A977953DF7ECBaB3CDb
WETHGateway: 0x9BdB5fcc80A49640c7872ac089Cc0e00A98451B6
Data Provider: 0x69FA688f1Dc47d4B5d8029D5a35FB7a548310654
IncentiveController: 0x929EC64c34a17401F460460D4B9390518E5B473e
Reward Token: 0x0000000000000000000000000000000000000000
Moola Strategy Encoded Params:  000000000000000000000000d05e3e715d945b59290df0ae8ef85c1bdb684744000000000000000000000000beadf48d62acc944a06eeae0a9054a90e5a7dc970000000000000000000000007551b5d2763519d4e37e8b81929d336de671d46d000000000000000000000000357d51124f59836ded84c8a1730d72b749d8bc230000000000000000000000000000000000000000000000000000000000000000


Constructor Arguments ABI-Encoded:
0000000000000000000000008f3cf7ad23cd3cadbd9735aff958023239c6a063000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000030000000000000000000000000000000000000000000000000000000000093a800000000000000000000000000000000000000000000000000000000000093a8000000000000000000000000000000000000000000000000029a2241af62c000000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000001ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff00000000000000000000000000000000000000000000000000000000000000000000000000000000000000007f8ba69d2d7bd4490ab0aa35b92e29b845aab7fa0000000000000000000000000000000000000000000000000000000000000000





   > Saving artifacts
   -------------------------------------
   > Total cost:         0.213315808 ETH
```


Summary
=======
> Total deployments:   3
> Final cost:          0.213315808 ETH


- **Aave V2 Strategy Based Pool** Start by setting the `MNEMONIC` var (which is the 12 word seed phrase in your wallet) & the `RPC` var in the .env file & then make sure you have the [right deployment configs set](https://github.com/Good-Ghosting/goodghosting-protocol-v1/blob/master/deploy.config.ts)(if a **whitelisted pool** needs to be deployed make sure the merkle root is set and the [isWhitelisted var](https://github.com/Good-Ghosting/goodghosting-protocol-v1/blob/master/deploy.config.ts#L135) is true), then just run `yarn deploy:polygon-aave`.

You will see something like this:

```
Starting migrations...
======================
> Network name:    'polygon-aave'
> Network id:      137
> Block gas limit: 30000000 (0x1c9c380)


2_deploy_contracts.js
=====================

   Deploying 'AaveStrategy'
   ------------------------
   > transaction hash:    0x3d0445201814629cf0eea2e68f0c034a288708ec62e5c50ef558f3fdff30b873
   > Blocks: 2            Seconds: 9
   > contract address:    0x7f8bA69d2D7bD4490AB0aa35B92e29B845aaB7fA
   > block number:        26186433
   > block timestamp:     1647862592
   > account:             0xf88b0247e611eE5af8Cf98f5303769Cba8e7177C
   > balance:             12.653319168891911123
   > gas used:            2004678 (0x1e96c6)
   > gas price:           32 gwei
   > value sent:          0 ETH
   > total cost:          0.064149696 ETH

   Pausing for 2 confirmations...
   ------------------------------
   > confirmation number: 3 (block: 26186439)

   Replacing 'SafeMath'
   --------------------
   > transaction hash:    0x651ba018aa6709ef01644b45b353111814630922d4ae03d976a7de4e8e37adc3
   > Blocks: 3            Seconds: 5
   > contract address:    0xAE130829ffeD8249BE3323289f15E4Bfd0770203
   > block number:        26186444
   > block timestamp:     1647862614
   > account:             0xf88b0247e611eE5af8Cf98f5303769Cba8e7177C
   > balance:             12.651008224891911123
   > gas used:            72217 (0x11a19)
   > gas price:           32 gwei
   > value sent:          0 ETH
   > total cost:          0.002310944 ETH

   Pausing for 2 confirmations...
   ------------------------------
   > confirmation number: 2 (block: 26186449)

   Replacing 'Pool'
   ----------------
   > transaction hash:    0xb16d515ed33d945d1c38be20c384d314ab0e129a602fc08eb44b4963f6bfcca1
   > Blocks: 5            Seconds: 10
   > contract address:    0x43a84D3BC0Fb6CFC93c7F9D08d8Be46a500bd9f3
   > block number:        26186454
   > block timestamp:     1647862634
   > account:             0xf88b0247e611eE5af8Cf98f5303769Cba8e7177C
   > balance:             12.504153056891911123
   > gas used:            4589224 (0x4606a8)
   > gas price:           32 gwei
   > value sent:          0 ETH
   > total cost:          0.146855168 ETH

   Pausing for 2 confirmations...
   ------------------------------
   > confirmation number: 2 (block: 26186461)



----------------------------------------------------
GoodGhosting Holding Pool deployed with the following arguments:
----------------------------------------------------

Network Name: polygon-aave
Contract's Owner: 0xf88b0247e611eE5af8Cf98f5303769Cba8e7177C
Inbound Currency: 0x8f3Cf7ad23Cd3CaDbD9735AFf958023239c6A063
Maximum Flexible Segment Payment Amount: 0
Segment Count: 3
Segment Length: 604800 seconds
Waiting Segment Length: 604800 seconds
Segment Payment: 3 dai (3000000000000000000 wei)
Early Withdrawal Fee: 1%
Custom Pool Fee: 1%
Max Quantity of Players: 115792089237316195423570985008687907853269984665640564039457584007913129639935
Flexible Deposit Pool: false
Transactional Token Depsoit Pool: false
Strategy: 0x7f8bA69d2D7bD4490AB0aa35B92e29B845aaB7fA
Lending Pool Provider: 0xd05e3E715d945B59290df0ae8eF85c1BdB684744
WETHGateway: 0xbEadf48d62aCC944a06EEaE0A9054A90E5A7dc97
Data Provider: 0x7551b5D2763519d4e37e8B81929D336De671d46d
IncentiveController: 0x357D51124f59836DeD84c8a1730D72B749d8BC23
Reward Token: 0x0000000000000000000000000000000000000000
Moola Strategy Encoded Params:  000000000000000000000000d05e3e715d945b59290df0ae8ef85c1bdb684744000000000000000000000000beadf48d62acc944a06eeae0a9054a90e5a7dc970000000000000000000000007551b5d2763519d4e37e8b81929d336de671d46d000000000000000000000000357d51124f59836ded84c8a1730d72b749d8bc230000000000000000000000000000000000000000000000000000000000000000


Constructor Arguments ABI-Encoded:
0000000000000000000000008f3cf7ad23cd3cadbd9735aff958023239c6a063000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000030000000000000000000000000000000000000000000000000000000000093a800000000000000000000000000000000000000000000000000000000000093a8000000000000000000000000000000000000000000000000029a2241af62c000000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000001ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff00000000000000000000000000000000000000000000000000000000000000000000000000000000000000007f8ba69d2d7bd4490ab0aa35b92e29b845aab7fa0000000000000000000000000000000000000000000000000000000000000000





   > Saving artifacts
   -------------------------------------
   > Total cost:         0.213315808 ETH
```


Summary
=======
> Total deployments:   3
> Final cost:          0.213315808 ETH


- **Curve Strategy Based Pool** Start by setting the `MNEMONIC` var (which is the 12 word seed phrase in your wallet) & the `RPC` var in the .env file & then make sure you have the [right deployment configs set](https://github.com/Good-Ghosting/goodghosting-protocol-v1/blob/master/deploy.config.ts)(if a **whitelisted pool** needs to be deployed make sure the merkle root is set and the [isWhitelisted var](https://github.com/Good-Ghosting/goodghosting-protocol-v1/blob/master/deploy.config.ts#L135) is true), then just run `yarn deploy:polygon-curve`.

You will see something like this:

```
Starting migrations...
======================
> Network name:    'polygon-curve'
> Network id:      137
> Block gas limit: 30000000 (0x1c9c380)


2_deploy_contracts.js
=====================

   Deploying 'CurveStrategy'
   -------------------------
   > transaction hash:    0xd74b8a02319bc7b83a212e31307865b87cca3370d97103308836fb2c81a9585b
   > Blocks: 5            Seconds: 9
   > contract address:    0x96083906aAD9dC0860e5B05a919f190213701Fae
   > block number:        26184454
   > block timestamp:     1647858510
   > account:             0xf88b0247e611eE5af8Cf98f5303769Cba8e7177C
   > balance:             3.060655209613386811
   > gas used:            2354830 (0x23ee8e)
   > gas price:           32 gwei
   > value sent:          0 ETH
   > total cost:          0.07535456 ETH

   Pausing for 2 confirmations...
   ------------------------------
   > confirmation number: 2 (block: 26184460)

   Deploying 'SafeMath'
   --------------------
   > transaction hash:    0xa0cb670b6e7cca13a7b1fb4a240144eecee6fdf78e3dfe5142980f09f1bc287d
   > Blocks: 5            Seconds: 9
   > contract address:    0x0a440F7B5414dFa7f5Ea94a01a21b8fC36f23276
   > block number:        26184468
   > block timestamp:     1647858538
   > account:             0xf88b0247e611eE5af8Cf98f5303769Cba8e7177C
   > balance:             3.058344265613386811
   > gas used:            72217 (0x11a19)
   > gas price:           32 gwei
   > value sent:          0 ETH
   > total cost:          0.002310944 ETH

   Pausing for 2 confirmations...
   ------------------------------
   > confirmation number: 2 (block: 26184474)

   Deploying 'Pool'
   ----------------
   > transaction hash:    0x76a240fc028f3b05d6a84fe160f3560dc991392a635257d7a102dcf8debc08d5
   > Blocks: 5            Seconds: 9
   > contract address:    0x032A14626522c502389EC5Eec1F3827697e1a0d1
   > block number:        26184482
   > block timestamp:     1647858566
   > account:             0xf88b0247e611eE5af8Cf98f5303769Cba8e7177C
   > balance:             2.911489097613386811
   > gas used:            4589224 (0x4606a8)
   > gas price:           32 gwei
   > value sent:          0 ETH
   > total cost:          0.146855168 ETH

   Pausing for 2 confirmations...
   ------------------------------
   > confirmation number: 3 (block: 26184488)



----------------------------------------------------
GoodGhosting Holding Pool deployed with the following arguments:
----------------------------------------------------

Network Name: polygon-curve
Contract's Owner: 0xf88b0247e611eE5af8Cf98f5303769Cba8e7177C
Inbound Currency: 0x8f3Cf7ad23Cd3CaDbD9735AFf958023239c6A063
Maximum Flexible Segment Payment Amount: 0
Segment Count: 3
Segment Length: 604800 seconds
Waiting Segment Length: 604800 seconds
Segment Payment: 3 dai (3000000000000000000 wei)
Early Withdrawal Fee: 1%
Custom Pool Fee: 1%
Max Quantity of Players: 115792089237316195423570985008687907853269984665640564039457584007913129639935
Flexible Deposit Pool: false
Transactional Token Depsoit Pool: false
Strategy: 0x96083906aAD9dC0860e5B05a919f190213701Fae
Curve Pool: 0x1d8b86e3d88cdb2d34688e87e72f388cb541b7c8
Curve Gauge: 0x3b6b158a76fd8ccc297538f454ce7b4787778c7c
Token index: 0
Pool Type: 1
Reward Token: 0x0d500B1d8E8eF31E21C99d1Db9A6444d3ADf1270
Curve Token: 0x172370d5cd63279efa6d502dab29171933a610af
Curve Strategy Encoded Params:  0000000000000000000000001d8b86e3d88cdb2d34688e87e72f388cb541b7c8000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000010000000000000000000000003b6b158a76fd8ccc297538f454ce7b4787778c7c0000000000000000000000000d500b1d8e8ef31e21c99d1db9a6444d3adf1270000000000000000000000000172370d5cd63279efa6d502dab29171933a610af


Constructor Arguments ABI-Encoded:
0000000000000000000000008f3cf7ad23cd3cadbd9735aff958023239c6a063000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000030000000000000000000000000000000000000000000000000000000000093a800000000000000000000000000000000000000000000000000000000000093a8000000000000000000000000000000000000000000000000029a2241af62c000000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000001ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff000000000000000000000000000000000000000000000000000000000000000000000000000000000000000096083906aad9dc0860e5b05a919f190213701fae0000000000000000000000000000000000000000000000000000000000000000





   > Saving artifacts
   -------------------------------------
   > Total cost:         0.224520672 ETH


Summary
=======
> Total deployments:   3
> Final cost:          0.224520672 ETH
```


## Celo
Start by setting the `MNEMONIC` var (which is the 12 word seed phrase in your wallet) in the .env file, also set the `CELO_PRIVATE_KEY` var which is the private key of your celo wallet & then make sure you have the [right deployment configs set](https://github.com/Good-Ghosting/goodghosting-protocol-v1/blob/master/deploy.config.ts)(if a **whitelisted pool** needs to be deployed make sure the merkle root is set and the [isWhitelisted var](https://github.com/Good-Ghosting/goodghosting-protocol-v1/blob/master/deploy.config.ts#L135) is true). Open two terminal windows, in one of them run
`ganache-cli -f https://forno.celo.org/ -m "clutchaptain shoe salt awake harvest setup primary inmate ugly among become" -i 999 --unlock {WHALE_ADDRESS_FORKED_NETWORK}` in the 2nd window run `yarn deploy:celo-mobius` for mobius strategy based pool or `yarn deploy:celo-moola` for moola strategy based pool.

You will see something like this:

```
Starting migrations...
======================
> Network name:    'celo-mobius'
> Network id:      42220
> Block gas limit: 0 (0x0)


2_deploy_contracts.js
=====================

Replacing 'MobiusStrategy'
--------------------------
> transaction hash:    0x258fd6b7586f385c8c9b0506a0a38b31411a88a9d5377375addb171399215093
> Blocks: 1            Seconds: 4
> contract address:    0x422Bf01090c47E0A5222A740433Eb6D7AEA4c328
> block number:        11985352
> block timestamp:     1647505428
> account:             0xf88b0247e611eE5af8Cf98f5303769Cba8e7177C
> balance:             2.110748391676738485
> gas used:            1856790 (0x1c5516)
> gas price:           0.5 gwei
> value sent:          0 ETH
> total cost:          0.000928395 ETH


Replacing 'SafeMath'
--------------------
> transaction hash:    0xdc5acfd4e69d3b5de2b6012af3cfa6fa057576b13d8bc83541c649ac97391198
> Blocks: 0            Seconds: 0
> contract address:    0xb2C98f7f573bbf653972F030766e36138C82F4A2
> block number:        11985353
> block timestamp:     1647505433
> account:             0xf88b0247e611eE5af8Cf98f5303769Cba8e7177C
> balance:             2.110712283176738485
> gas used:            72217 (0x11a19)
> gas price:           0.5 gwei
> value sent:          0 ETH
> total cost:          0.0000361085 ETH


Replacing 'Pool'
----------------
> transaction hash:    0x16194fe3dda5f6fe40f98a36a4bbca24656c38853e7210fb57c2551e1e26df7f
> Blocks: 0            Seconds: 0
> contract address:    0x99E91F09991966aBe0DC59555a5C1e25a78E08B7
> block number:        11985354
> block timestamp:     1647505438
> account:             0xf88b0247e611eE5af8Cf98f5303769Cba8e7177C
> balance:             2.108424171176738485
> gas used:            4576224 (0x45d3e0)
> gas price:           0.5 gwei
> value sent:          0 ETH
> total cost:          0.002288112 ETH




----------------------------------------------------
GoodGhosting Holding Pool deployed with the following arguments:
----------------------------------------------------

Network Name: celo-mobius
Contract's Owner: 0xf88b0247e611eE5af8Cf98f5303769Cba8e7177C
Inbound Currency: 0x765DE816845861e75A25fCA122bb6898B8B1282a
Maximum Flexible Segment Payment Amount: 0
Segment Count: 3
Segment Length: 604800 seconds
Waiting Segment Length: 604800 seconds
Segment Payment: 3 dai (3000000000000000000 wei)
Early Withdrawal Fee: 1%
Custom Pool Fee: 1%
Max Quantity of Players: 115792089237316195423570985008687907853269984665640564039457584007913129639935
Flexible Deposit Pool: false
Transactional Token Depsoit Pool: false
Strategy: 0x422Bf01090c47E0A5222A740433Eb6D7AEA4c328
Mobius Pool: 0x9906589Ea8fd27504974b7e8201DF5bBdE986b03
Mobius Gauge: 0xc96AeeaFF32129da934149F6134Aa7bf291a754E
Mobius Minter: 0x5F0200CA03196D5b817E2044a0Bb0D837e0A7823
Mobi Token: 0x73a210637f6F6B7005512677Ba6B3C96bb4AA44B
Celo Token: 0x471EcE3750Da237f93B8E339c536989b8978a438
Mobius Strategy Encoded Params:  0000000000000000000000009906589ea8fd27504974b7e8201df5bbde986b03000000000000000000000000c96aeeaff32129da934149f6134aa7bf291a754e0000000000000000000000005f0200ca03196d5b817e2044a0bb0d837e0a782300000000000000000000000073a210637f6f6b7005512677ba6b3c96bb4aa44b000000000000000000000000471ece3750da237f93b8e339c536989b8978a438


Constructor Arguments ABI-Encoded:
000000000000000000000000765de816845861e75a25fca122bb6898b8b1282a000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000030000000000000000000000000000000000000000000000000000000000093a800000000000000000000000000000000000000000000000000000000000093a8000000000000000000000000000000000000000000000000029a2241af62c000000000000000000000000000000000000000000000000000000000000000000010000000000000000000000000000000000000000000000000000000000000001ffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffffff0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000422bf01090c47e0a5222a740433eb6d7aea4c3280000000000000000000000000000000000000000000000000000000000000000





> Saving artifacts
-------------------------------------
> Total cost:        0.0032526155 ETH


Summary
=======
> Total deployments:   3
> Final cost:          0.0032526155 ETH
```





### Merkle Root Generation for Whitelisted Contracts
To deploy the `WhitelistedPool` contract, a merkle root is required, introduced for the purpose of whitelisting users. The merkle root can be created by using the repo below:

Clone this [repository](https://github.com/Good-Ghosting/Whitelisting)

Install Dependencies: `yarn install`

Edit this [file](https://github.com/Good-Ghosting/goodghosting-whitelisting/blob/master/scripts/input.csv) with the addresses you want to whitelist, keeping the JSON format same.

Run: `yarn generate-merkle-root`

You should see an output similar to this:

`{
  "merkleRoot": "0xc65049d2040e43b130c923276515ed14d241ac88d28f0c03384d5b5f7197be82",
  "claims": {
    "0xBE73748446811eBC2a4DDdDcd55867d013D6136e": {
      "index": 0,
      "exists": "true",
      "proof": ["0x1f122d8c45929e68268031d8ce59ea362ab716d6b93f3b226c4cdcf459c766b3"]
    },
    "0xb9a28ce32dcA69AD25E17212bC6D3D753E795aAe": {
      "index": 1,
      "exists": "true",
      "proof": ["0xdbab8c7f829217c06dc0a73baaefdbfd9e15c463a255e1b3947b27f7792462de"]
    }
  }
}`

Copy the value of the `merkleRoot` field, and replace the merkle root parameter in the [deploy.config.js](https://github.com/Good-Ghosting/goodghosting-protocol-v1/blob/master/deploy.config.ts) file. Once this step is done, the contract can be deployed using the deployment instructions provided above.
