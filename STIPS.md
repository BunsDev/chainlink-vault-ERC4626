# Spec Doc for CL Vault

## Abstract
Interacting with bridges is challenging and imposes a cost and time commitment on the user that is a sub-standard experience relative to traditional asset management
## Motivation
### Feature
- **Description**: An ERC4626 vault that can execute bridging and investments on behalf of the user, using CCIP as the underlying messaging protocol.

### Why is this feature necessary?
- To make yield opportunities available to users on their preferred chain.

### Who is this feature for?
- **Managers**: Create vaults that include a more diverse set of assets and strategies.
- **Investors**: Improved UI for allocating to opportunities beyond your preferred chain.

### When and how is this feature going to be used?
- Used anytime a manager wants to abstract away cross-chain interactions from their users.

### User Story
Sandra has all of her funds on Base. She is a new DeFi user and just got a metamask account. She has heard about good yield on Avalanche AVAX, but is unsure about bridges and the idea of multiple wallets and chains.

X-Chain Yield Vaults provide a way that she can easily deposit collateral into her preferred chain, and on the backend it handles all of the bridging for her. For example, she deposits ETH on Base and receives a share token that represents her claim on Staked AVAX on Avalanche.

## Background Information

### ERC4626
ERC4626 is the standardized vault contract used for accounting and managing user withdrawals and deposits. 

#### Examples & Contracts:
- [Solmate](https://github.com/transmissions11/solmate/blob/main/src/mixins/ERC4626.sol)
- [Open Zeppelin](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/extensions/ERC4626.sol)
- A simple implementation on [QuickNode](https://www.quicknode.com/guides/ethereum-development/smart-contracts/how-to-use-erc-4626-with-your-smart-contract#what-you-will-need)
- Smart Contract Programmer [Vault Math](https://youtu.be/k7WNibJOBXE?si=kwVLuDNLKkWEQ1cc)

#### Accounting:
- **Deposit Asset**: Use CCIP-BnM on source chain as Asset A. All the vault accounting on the source chain will be in this asset
- **Yield Asset**: Use (Create an ERC20) on destination chain as Asset B. _Question: should the vault on the destination chain be in this asset or Asset A?_
- **Total Assets Calculation**: `totalAssets()` is defined in wAVAX according to the total of both and exchange rate between them.
- **Fetching Timestamp**: Call `historicalExchangeRateTimestamps(0)` to get `_timestamp`.
- **Getting Exchange Rate**: Call `historicalExchangeRateByTimestamps(_timestamp)` to get rate in wei.
- **Handling Execution Data**:
  - The challenge is obtaining execution data on swaps (slippage) and exchange rate back to the vault contracts on the destination chain to ensure the `totalAssets()` call is accurate.
  - Use CCIP for this purpose.
- **Keeper Role**:
  - A keeper grabs the exchange rate and execution data from the swap and returns it to the destination chain using CCIP.
  - Keeper on homechain executes a function to update `totalAssets()` based on this data.
- **Redemption Process**:
  - User shares on redeem are defined as their percentage of `totalSupply`.
  - User initiates a redeem call.
  - Calculate `user_withdrawal_perc = number of shares / totalSupply()`.
  - Determine the amount to transfer out: `amount to transfer out = user_withdrawal_perc * totalAssets()`.

### Chainlink Automation
- **Getting Started**: [Guide](https://docs.chain.link/chainlink-automation/overview/getting-started)
- **Better Guide**: [Time-based upkeeps](https://docs.chain.link/quickstarts/time-based-upkeep)
  - Time-based upkeeps are smart contracts set up using Chainlink Automation
      - Allow the automatic execution of a smart contract function on a custom schedule
      - Similar to cron jobs
      - Can be used to trigger smart contract functions.
  - Register a new upkeep on [automation.chain.link](automation.chain.link).
  - Use a cron expression to set the time interval. For example, `0 0 * * *` runs at midnight every 24 hours.
  - Consider setting upkeep to call some bridge function on the vault smart contract.
- **Custom Logic**: 
  - Import the right Chainlink libraries to set custom logic within your smart contract.
  - For Example, topping up a contract when the balance falls too low
  - Guide on Chainlink keeper: [CL Keeper - Guide](https://docs.chain.link/chainlink-automation/guides/compatible-contracts)
- **Log-based Automation**:
    - Chainlink Automation offers a feature called Log Trigger Upkeep, which allows you to monitor specific events like deposits on a vault and trigger actions based on them. This feature is useful for automating responses to on-chain events without continuous manual monitoring.

    - To use this, you need to implement the ILogAutomation interface in your smart contract. This involves defining functions like checkLog and performUpkeep. checkLog is used to parse log data and check if an on-chain action is needed, while performUpkeep executes the necessary on-chain actions.

    - You can deploy a contract with this interface, such as CountWithLog, which uses events to trigger actions. Then, you can register your contract with Chainlink Automation, specifying the details and conditions under which your contract should react to logs. Once set up, Chainlink Automation will monitor the logs and execute your contract's functions when the specified conditions are met.

    - For detailed steps and examples, you can refer to the specific Chainlink documentation or guides available online. This feature streamlines processes for smart contracts, making them more responsive and efficient.

### Chainlink CCIP
- Install Foundry Chainlink Toolkit: `forge install smartcontractkit/foundry-chainlink-toolkit`
- Install CCIP Node Package: `npm install @chainlink/contracts-ccip --save`
- Supported [Testnets](https://docs.chain.link/ccip/supported-networks/testnet)
- **Uniswap V2 [Deployment](https://sepolia.etherscan.io/address/0xC532a74256D3Db42D0Bf7a0400fEFDbad7694008#code)** on Sepolia
- **[CCIP Test Tokens](https://docs.chain.link/ccip/test-tokens#mint-tokens-in-the-documentation)**:
  - **BnM**: These tokens are minted on each testnet. When transferring these tokens between testnet blockchains, CCIP burns the tokens on the source chain and mints them on the destination chain.
  - **LnM**: These tokens are only minted on Ethereum Sepolia. On other testnet blockchains, the token representation is a wrapped/synthetic asset called clCCIP-LnM. When transferring these tokens from Ethereum Sepolia to another testnet, CCIP locks the CCIP-LnM tokens on the source chain and mints the wrapped representation clCCIP-LnM on the destination chain. Between non-Ethereum Sepolia chains, CCIP burns and mints the wrapped representation clCCIP-LnM.
 
- [**Masterclass**](https://andrej-rakic.gitbook.io/chainlink-ccip/ccip-masterclass/exercise-1-transfer-tokens) - use this guide to design your contracts
- **API References** - use these to construct your function calls
  - [RouterClient](https://docs.chain.link/ccip/api-reference/i-router-client#ccipsend)
  - [CCIPReceiver](https://docs.chain.link/ccip/api-reference/ccip-receiver)
  - [Client Library](https://docs.chain.link/ccip/api-reference/client)
- **IRouterClient** Import statement:
```solidity
import {IRouterClient} from "@chainlink/contracts-ccip/src/v0.8/ccip/interfaces/IRouterClient.sol";
...
IRouterClient router;
constructor(address _router) {
     router = IRouterClient(_router);
 }
```
- **ccipSend** - request a message to be sent to the destination chain
```solidity
function ccipSend(uint64 destinationChainSelector, struct Client.EVM2AnyMessage message) external payable returns (bytes32)
```
- **EVM2AnyMessage** - CCIP senders use this solidity struct to build the CCIP message.
```solidity
struct EVM2AnyMessage {
  bytes receiver;
  bytes data;
  struct Client.EVMTokenAmount[] tokenAmounts;
  address feeToken;
  bytes extraArgs;
}
```
 
- **Any2EVMMessage** - CCIP receivers use this solidity struct to parse the received CCIP message.
```solidity
struct Any2EVMMessage {
  bytes32 messageId;
  uint64 sourceChainSelector;
  bytes sender;
  bytes data;
  struct Client.EVMTokenAmount[] destTokenAmounts;
}
```

| Name         | Type                       | Description                                                                                       |
|--------------|----------------------------|---------------------------------------------------------------------------------------------------|
| receiver     | bytes                      | Receiver address. Use abi.encode(sender) to encode the address to bytes.                          |
| data         | bytes                      | Payload sent within the CCIP message.                                                             |
| tokenAmounts | Client.EVMTokenAmount[]    | Tokens and their amounts in the source chain representation.                                      |
| feeToken     | address                    | Address of feeToken. Set address(0) to pay in native gas tokens such as ETH on Ethereum or MATIC on Polygon. |
| extraArgs    | bytes                      | Users fill in the EVMExtraArgsV1 struct then encode it to bytes using the _argsToBytes function.  |

- extraArgs
```solidity
bytes4 EVM_EXTRA_ARGS_V1_TAG
```
- EVMExtraArgsV1
```solidity
struct EVMExtraArgsV1 {
  uint256 gasLimit;
  bool strict;
}
```

| Name     | Type    | Description                                                                                              |
|----------|---------|----------------------------------------------------------------------------------------------------------|
| gasLimit | uint256 | Specifies the maximum amount of gas CCIP can consume to execute ccipReceive() on the contract located on the destination blockchain. Read Setting gasLimit for more details. |
| strict   | bool    | Used for strict sequencing. Read Sequencing for more details.                                            |

#### Simple Contracts in Foundry - [Examples](https://github.com/smartcontractkit/ccip-starter-kit-foundry/tree/main/src) 

- **BasicMessageReceiver**
- **BasicMessageSender**
- **BasicTokenSender**
- **[CCIPReceiver.sol](https://github.com/0xCSMNT/ccip-testing/blob/master/node_modules/%40chainlink/contracts-ccip/src/v0.8/ccip/applications/CCIPReceiver.sol)** - Abstract contract. Use this to receive messages on the destination chain, inherit from npm toolkit.
- **[Transferor.sol](https://github.com/0xCSMNT/ccip-testing/blob/master/Transferor.sol)** - Example contract of a simple transferring contract on the home chain, with features like whitelisting.

#### Interaction with [CCIP Architecture](https://docs.chain.link/ccip/architecture)

##### The Router
- The primary contract CCIP users interface with.
- Responsible for initiating cross-chain interactions. One router contract exists per chain.
- When transferring tokens, callers must approve tokens for the router contract.
- Routes the instruction to the destination-specific OnRamp.

##### Commit Store
- The Committing DON interacts with the CommitStore contract on the destination blockchain to store the Merkle root of the finalized messages on the source blockchain.

##### OnRamp
- One OnRamp contract per lane.
- Checks destination-blockchain-specific validity (e.g., validating account address syntax).
- Verifies the message size limit and gas limits.
- Keeps track of sequence numbers to preserve the sequence of messages for the receiver.
- Manages billing and interacts with the TokenPool if the message includes a token transfer.
- Emits an event monitored by the committing DON.

##### OffRamp
- One OffRamp contract per lane.
- Ensures message authenticity by verifying the proof against a committed and blessed Merkle root.
- Executes transactions only once.
- After validation, transmits received messages to the Router contract.
- If the CCIP transaction includes token transfers, calls the TokenPool to transfer assets to the receiver.

##### Token Pools
- Each token has its own token pool, an abstraction layer over ERC-20 tokens facilitating OnRamp and OffRamp operations.
- Configurable to lock or burn at the source blockchain and unlock or mint at the destination blockchain.

### Uniswap V2 
#### Example of a swap after a CCIP message is received 
Note: created by ChatGPT, so probably inaccurate.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import { IRouterClient } from "@chainlink/contracts-ccip/src/v0.8/ccip/interfaces/IRouterClient.sol";
import { CCIPReceiver } from "@chainlink/contracts-ccip/src/v0.8/ccip/applications/CCIPReceiver.sol";
import { Client } from "@chainlink/contracts-ccip/src/v0.8/ccip/libraries/Client.sol";
import "@uniswap/v2-core/contracts/interfaces/IUniswapV2Factory.sol";
import "@uniswap/v2-core/contracts/interfaces/IUniswapV2Pair.sol";
import "@uniswap/v2-periphery/contracts/interfaces/IUniswapV2Router02.sol";

contract CCIPSwapReceiver is CCIPReceiver {
    IUniswapV2Router02 public uniswapRouter;
    IRouterClient public ccipRouter;
    address public linkToken;

    constructor(address _ccipRouter, address _uniswapRouter, address _linkToken) CCIPReceiver(_ccipRouter) {
        uniswapRouter = IUniswapV2Router02(_uniswapRouter);
        ccipRouter = IRouterClient(_ccipRouter);
        linkToken = _linkToken;
    }

    function _ccipReceive(Client.Any2EVMMessage memory message) internal override {
        // Decode the message to extract the swap details
        (address tokenToSwap, uint amountIn, uint amountOutMin, address[] memory path, uint64 sourceChain, bytes memory returnData) = abi.decode(
            message.data,
            (address, uint, uint, address[], uint64, bytes)
        );

        // Transfer the tokens from the sender to this contract
        IERC20(tokenToSwap).transferFrom(msg.sender, address(this), amountIn);

        // Approve the Uniswap router to spend the tokens
        IERC20(tokenToSwap).approve(address(uniswapRouter), amountIn);

        // Perform the swap on Uniswap and capture the output amount
        uint[] memory amounts = uniswapRouter.swapExactTokensForTokens(
            amountIn,
            amountOutMin,
            path,
            address(this),
            block.timestamp + 15 minutes // deadline
        );

        // Build and send the CCIP message back to the original chain
        _sendCCIPResponse(sourceChain, amounts[amounts.length - 1], returnData);
    }

    function _sendCCIPResponse(uint64 sourceChain, uint256 swapOutput, bytes memory returnData) internal {
        // Encode the swap output data
        bytes memory data = abi.encode(swapOutput, returnData);

        // Create an EVM2AnyMessage
        Client.EVM2AnyMessage memory ccipMessage = Client.EVM2AnyMessage({
            receiver: data,
            data: "",
            tokenAmounts: new Client.EVMTokenAmount[](0), // No token transfer in this message
            extraArgs: Client._argsToBytes(Client.EVMExtraArgsV1({gasLimit: 200_000, strict: false})),
            feeToken: linkToken
        });

        // Calculate the fee and ensure this contract has enough LINK to cover it
        uint256 fee = ccipRouter.getFee(sourceChain, ccipMessage);
        require(IERC20(linkToken).balanceOf(address(this)) >= fee, "Insufficient LINK for CCIP fee");

        // Approve the Router to spend LINK
        IERC20(linkToken).approve(address(ccipRouter), fee);

        // Send the CCIP message
        ccipRouter.ccipSend(sourceChain, ccipMessage);
    }
}

```

## Open Questions
- [x] What assets and what chain to use? Why?
    - *Source Vault Chain: Avalanche Fuji*
    - *Destination Vault Chain: Ethereum Sepolia*
      - *This ensures the fastest settlement*
    - *Deposit Asset on Source Chain: CCIP-BnM*
    - *Yield Asset on Destination Chain: Create a basic ERC20 on Sepolia*
      -  *This ensures we have a bridgeable asset and access to a [Uni V2 Fork](https://sepolia.etherscan.io/address/0xC532a74256D3Db42D0Bf7a0400fEFDbad7694008#code) for swaps on the destination chain* 
- [ ]How will the swap on the destination chain trigger the CCIP message to update the accounting?
      - *Part of a _ccipReceive() function call* 
        - ~~The whole purpose of accounting is to issue the right amount of shares, so if a new user deposits on the source chain after the above swap, the NAV of the vault will be calculated based on the post swap asset balances. Our goal is to be able to get this value across to the source chain using a function call on the source chain.~~
      - _This is not right becase the deposit vault is unaware of NAV decay from the swap until that data has been returned by CCIP_
  Should the origin vault also be the sender contract and the destination vault be the reciever contract?
      - OR Should the token transfers be routed via a protocol controlled transfer contract?
        - Same contract
      - Also can deposit + bridging be atomic ie in the same transaction? What is the expected latency?
      - Latency is equal to block confirmation time
- [ ] What testnet deployments are ruled out by picking specific technologies
    - *For example: Li.Fi is not on Sepolia*
- [ ] Returning execution **data from the swap transaction** on the destination chain: Are we sure that can be handled in the same function call as the swap itself? do we not need to wait for a block so we have something to read from? 
    - *Reasonably confident this is possible*
- [ ] For withdrawals: Are we sure we want **push instead of pull** for transfering assets to user?
    - pull is safe lets do that
- [ ] How do we enable pull on the source chain vault? How do we keep user funds safer
    - Use a mapping to set withdrawal limits to 0 for all users while a withdrawal & bridge action is in place

## Feasibility Analysis
Provide potential solution(s) including the pros and cons of those solutions and who are the different stakeholders in each solution. A recommended solution should be chosen here. A combination of the below solutions will be used for accomplishing the goals of the project.

### Options Considered 
- ~~**Bridging Solution A** - Use Li.Fi to bridge, and native destination chain swap~~
- **Bridging Solution B** - Use Chainlink CCIP to send tokens and then use native destination chain swap
- ~~**Custody Solution A** - User Funds on Destination Chain sit in an EOA and cannot be redeemed (one way trip)~~
- **Custody Solution B** - User Funds on Destination Chain are in a separate vault that users can withdraw from (much more complicated but possible with CCIP I think)
- **Locking Option** - Vault is locked during bridging and swapping sequence to protect against attacks

### Decision on High Level Design Plan
The chosen design will use CCIP as the underlying messaging protocol. As there are a limited number of tokens enabled for the protocol to date, CCIP-BnM will be used as Asset A on the source chain vault, and that will be bridged to the Destination Chain Vault. Here is will be swapped to an ERC20 using a Uniswap V2 fork and deposited into another ERC4626.

The vault will be locked during bridging periods to protect from griefing while the value of the underlying is undefined.

## Checkpoint 1
Before more in depth design of the contract flows lets make sure that all the work done to this point has been exhaustive. It should be clear what we're doing, why, and for who. All necessary information on external protocols should be gathered and potential solutions considered. At this point we should be in alignment with product on the non-technical requirements for this feature. It is up to the reviewer to determine whether we move onto the next step.

## Requirements

### Source Vault

#### Expected Functionality
1. Accept user funds via a `deposit()` function.
2. Issue shares to user via `mint()`.
3. To issue shares, we need a method to calculate total value of the destination vault + source vault (if funds ever sit idle in the source vault).
4. We need the ERC20 method which returns `totalSupply`.
5. We need a `transfer()` method (or could keep it non-transferrable).
6. We need a method which can convert users LP token value to underlying asset value.
7. We need a method to invoke a CCIP transfer via relevant router.
8. We need to be able to check the status of the deposit on the destination chain.
9. A `withdraw()` function to accept user withdraw request.
10. A mapping to track if users withdraw request has been fulfilled.
11. A method to burn user shares.
12. A method to `lockVault()`
13. A method to `unlockVault()`

### Destination Vault

#### Expected Functionality
1. Accept a CCIP transfer.
2. Swap funds to target yield asset.
3. Swap funds back to base asset.
4. Return a value of assets in the vault in terms of base asset.
5. Accept a withdrawal request from the source vault.
6. Calls `swap()`.
7. Send a CCIP programmable token transfer as well as the message ID of the initially received message.
8. Swap funds based on the amount of requested funds.

### Deposits & Withdrawals Transaction Flows

##### Deposits & Daily Bridging 1

![3.png](assets/3.png)

##### Deposits & Daily Bridging 2

![4.png](assets/4.png)

##### Withdrawals 1

![5.png](assets/5.png)

##### Withdrawals 2

![6.png](assets/6.png)

##### Withdrawal Accounting


## Checkpoint 2
Before we spec out the contract(s) in depth we want to make sure that we are aligned on all the technical requirements and flows for contract interaction. Again the who, what, when, why should be clearly illuminated for each flow. It is up to the reviewer to determine whether we move onto the next step.

**Reviewer**:

Reviewer: []
## Specification

### Explanation

The chosen design will use CCIP as the underlying messaging protocol. As there are a limited number of tokens enabled for the protocol to date, CCIP-BnM will be used as Asset A on the source chain vault, and that will be bridged to the Destination Chain Vault. Here this will be swapped to an ERC20 of our creation using a Uniswap V2 fork and deposited into another ERC4626.

The vault will be locked during bridging periods to protect from griefing while the value of the underlying is undefined.

A third contract ExitVault will be deployed as an exit queue where users will access their withdrawn funds after a bridging event has been executed to meet their withdrawal request.

### Onchain Spec

- 3 Smart Contracts
- 1 Chainlink Automation Keeper 
- 1 Uni V2 Pool 

| Contract         | Chain         | Core Function                         |
|------------------|---------------|---------------------------------------|
| SourceVault.sol  | Fuji Testnet  | Deposit Asset Accounting & CCIP       |
| ExitVault.sol    | Fuji Testnet  | Holding account for withdrawals       |
| DestinationVault.sol | Sepolia Testnet | Yield Asset Accounting, Uni Swaps & CCIP |


### SourceVault

#### Inheritance & Imports
- `OwnerIsCreator` from Chainlink CCIP
- `IRouterClient` from Chainlink CCIP
- `IERC20` from OpenZeppelin
- `Client` from Chainlink CCIP
- `LinkTokenInterface` from Chainlink Contracts
- `ERC4626` from Solmate

#### Structs
(Define any custom structs that are needed for the contract.)

#### Constants
(Define any constants that are used within the contract.)

#### Constructor
- Arguments for the constructor should include addresses for the router, LINK token, and destination vault.
- Initializes `router`, `linkToken`, and sets the destination vault address.

#### State Variables
- `IRouterClient router`: Instance of IRouterClient for cross-chain communication.
- `LinkTokenInterface linkToken`: Instance of LinkTokenInterface to interact with LINK tokens.
- `mapping(uint64 => bool) public whitelistedChains`: Tracks the chains that are allowed for cross-chain communication.
- `Address destinationVault`: Address of the destination vault contract.
- `uint256 totalAssets`

#### Functions
- `deposit(uint256 amount)`: Accepts user deposit and updates internal accounting.
- `transferTokens(uint64 _destinationChainSelector, address _receiver, address _token, uint256 _amount)`: Transfers tokens and executes a function on the destination chain.
- `updateAccounting()`: Updates `totalAssets` based on cross-chain messages and internal logic.
- `whitelistChain(uint64 _chainId)`: Adds a chain to the whitelist.
- `addExitVault(address _exitVault)`: Links an ExitVault contract to this vault.
- `denylistChain(uint64 _chainId)`: Removes a chain from the whitelist.
- `requestWithdrawal(address _asset, uint256 _amount)`: Initiates a withdrawal process that may involve cross-chain actions.
- `_ccipReceive(bytes _message)`: Receives a message from the CCIP router and performs accounting updates.
- `exitAndUpdate() external onlyExitVault`: Allows the ExitVault contract to trigger accounting updates.
- `lockVault() external onlyDestinationVault`: Locks the vault to prevent new deposits during certain operations.
- `unlockVault() external onlyDestinationVault`: Unlocks the vault to allow new deposits.

#### Events
- `TokenBridged(address indexed token, uint256 amount)`: Emitted when tokens are successfully bridged.
- `AccountingUpdated(uint256 totalAssets)`: Emitted when the accounting is updated.

#### Modifiers
- `onlyWhitelistedChains(uint64 _chainId)`: Ensures function can only be called for whitelisted chains.
- `onlyDestinationVault()`: Ensures function can only be called by the destination vault.
- `onlyExitVault()`: Ensures function can only be called by the exit vault.

#### requestWithdrawal() Function Logic

- **User calls `requestWithdrawal()` function**
  - **If enough funds are available on the source chain side:**
    - User gets their tokens back and burns their shares (standard ERC4626 process).
  - **If not enough funds are available:**
    - `DepositVault` contract calls a `bridgeAndWithdraw()` function that:
      - Requests the required tokens back from CCIP, based on the user's proportion of the Net Asset Value (NAV).
      - Adds the user address to the `withdrawalLimit` mapping in `ExitVault.sol` with a default of zero.
    - `_ccipReceive()` on the destination chain receives the request for a proportion of tokens (not an absolute amount), and then:
      - Executes a swap for the user's proportional amount of the deposit asset.
      - Sends the tokens back to the `ExitVault` contract.
      - Calls an `updateWithdrawalLimit()` function on `ExitVault` that:
        - Sets the mapping for the user address to the same value as the number of tokens in the bridging transaction.
      - Calls an `updateVaultAssets()` function on `ExitVault` that:
        - Has external access to update the accounting on `SourceVault`.

### ExitVault 

#### Imports & Inheritance

- Inherits `OwnerIsCreator` from CCIP for ownership management.
- Inherits `IRouterClient` from CCIP for cross-chain communication.
- Uses `IERC20` from OpenZeppelin for ERC20 token interactions.
- Uses `Client` from CCIP for client-side functionalities.
- `LinkTokenInterface` for interacting with Chainlink's LINK tokens. (Note: Requires manual import as it is missing from npm install)

#### Structs

*(No structs specified. Add if any are used within the contract.)*

#### Constants

*(Specify any constants used within the contract here.)*

#### Constructor

*(Constructor details needed, such as initial setup of state variables.)*

#### State Variables

- `IRouterClient router`: Instance of IRouterClient for cross-chain communication.
- `mapping(uint64 => bool) public whitelistedChains`: Tracks which chains are approved for cross-chain communication.
- `mapping(address => uint256) public withdrawalLimits`: Records the withdrawal limits for each address.

#### Functions

- `_ccipReceive(bytes memory _data)`: Handles incoming CCIP messages and performs actions based on their content.
- `whitelistChain(uint64 _chainId)`: Authorizes a chain for cross-chain interactions.
- `whitelistSender(address _sender)`: Authorizes a sender for initiating transactions.
- `withdrawFromExitVault(address _beneficiary, uint256 _amount)`: Allows withdrawals from the exit vault up to the withdrawal limit.
- `updateSourceVaultAccounting()`: Updates the accounting details in the SourceVault contract.

#### Events

- `TokenReceived(address indexed sender, uint256 amount)`: Emitted when tokens are received in the vault.
- `TokenWithdrawn(address indexed beneficiary, uint256 amount)`: Emitted when tokens are withdrawn from the vault.

#### Modifiers

- `onlyWhitelistedChains(uint64 _chainId)`: Restricts certain actions to approved chains.
- `onlyWhitelistedSenders(address _sender)`: Restricts certain actions to approved senders.

*(For any gaps in the information, such as specific function arguments, return types, events, and internal logic, consultation with the smart contract development team for comprehensive details is recommended.)*

### DestinationVault

#### Inheritance
*(Inheritances details needed. List all base contracts that DestinationVault inherits from.)*

#### Structs
*(Structs details needed if any are used within the contract.)*

#### Constants
*(Constants details needed. Define any constant values used within the contract.)*

#### Constructor
*(Constructor details needed, such as initial setup of state variables.)*

#### Public Variables
- `address ExitVault`: Address of the associated ExitVault.
- `address SourceVault`: Address of the associated SourceVault.

#### Functions
- `transferTokensBack(address _token, uint256 _amount, address _to)`: Transfers tokens back to the ExitVault via CCIP.
- `transferDataBack(bytes memory _data)`: Sends value of assets from swap back to SourceVault via CCIP.
- `executeDepositSwap(...)`: *(Arguments and description needed)*
- `executeWithdrawalSwap(...)`: *(Arguments and description needed)*
- `whitelistChain(uint64 _chainId)`: Authorizes a chain for cross-chain interactions.
- `addExitVault(address _exitVault)`: Sets the address of the ExitVault.
- `addSourceVault(address _sourceVault)`: Sets the address of the SourceVault.
- `denylistChain(uint64 _chainId)`: Revokes authorization for a chain.
- `_ccipReceive(bytes memory _data)`: Handles incoming CCIP messages.

#### Events
- `TokensReceived(address indexed from, uint256 amount)`: Logs the receipt of tokens.
- `SwapExecuted(address indexed tokenIn, address indexed tokenOut, uint256 amountIn, uint256 amountOut)`: Logs execution of a swap.
- `SwapDataReturnedToSource(bytes data)`: Logs the swap data returned to the SourceVault.

#### Modifiers
- `onlyWhitelistedChain(uint64 _chainId)`: Restricts certain actions to approved chains.
- `onlySourceVault()`: Restricts certain actions to the SourceVault.


**Reviewer**:

## Implementation
[Link to implementation PR]()
## Documentation
[Link to Documentation on feature]()
## Deployment
[Link to Deployment script PR]()  
[Link to Deploy outputs PR]()

# Additional Research

## Transaction FLows
### CCIP/Messages/Transaction Flow/Functionality Mirror

| Action                                            | Source Action                                                                    | Destination action                                                                                     |
|---------------------------------------------------|----------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| User deposits                                     | Deposit()                                                                        | No action                                                                                             |
| Vault mints shares                                | mint()                                                                           | No action                                                                                             |
| totalAssets()                                     | Check for pre-updated value of destination assets, calculate present value in [base asset via Chainlink oracle](https://github.com/smartcontractkit/ccip-defi-lending/blob/c12632b6f1b0954a081e8c658b64ebbd81c4d980/contracts/Protocol.sol#L107) | Should pre-update values after each deposit in the destination asset. Default would be zero.          |
| After mint(), funds sit in the destination vault until bridging event called by CL keeper | A [sendFunds()](https://github.com/smartcontractkit/ccip-defi-lending/blob/c12632b6f1b0954a081e8c658b64ebbd81c4d980/contracts/Sender.sol#L71C17-L71C17) function constructs the message, initializes the router, sends the message, locks the vault and returns the messageId. | *A _ccipRecieve function identifies the message and uses the token amount to call a swap() function   |
|                                                   | A _ccipReceive function identifies the message and uses the received value to update the `totalAssets` on the vault, and call `unlockVault()` | A chainlink keeper uses a [log-based trigger](https://medium.com/@warissara.0039/log-trigger-upkeep-with-chainlink-automation-9d1805a29eda) to view the received tokens from the swap and return that data via CCIP to the source chain. |
| User requests withdraw                            | Withdraw()                                                                       | No action                                                                                             |
| Vault burns the shares, requests for funds from destination | Vault calculates base asset to be requested based on oracle value and on value of deposit asset currently in the source chain vault, burns the shares, records the value of base asset to be requests, user address, [requests for funds via CCIP](https://github.com/smartcontractkit/ccip-liquidation-protector/blob/f0e71131a6171ffe04deeec653b5d5efe9f3713f/contracts/monitors/MonitorCompoundV3.sol#L82) | Destination vault then checks for the amount needed, [swaps to the base token, and sends a CCIP token transfer](https://github.com/smartcontractkit/ccip-liquidation-protector/blob/f0e71131a6171ffe04deeec653b5d5efe9f3713f/contracts/LPSC.sol#L58C20-L58C20) |
| Vault receives funds, completes withdrawal request | __ccipreceive and then return funds to user # Spec Doc for CL Vault

Reference
[ccip-liquidation-protector](https://github.com/smartcontractkit/ccip-liquidation-protector/tree/f0e71131a6171ffe04deeec653b5d5efe9f3713f)

## Unused Protocol Research
### Chainlink Functions
- Call any API from a smart contract: Enables access to off-chain data and computation.
  - [Overview](https://chain.link/functions) and [docs](https://docs.chain.link/chainlink-functions)
  - Good [Video](https://youtu.be/I-g1aaZ3_x4?si=gKw8ccZS5__Kj0mD0) to get up to speed
  - Could be useful for interacting with **Li.Fi API**


### Li.Fi
Li.FI is a multichain bridge and DEX aggregator with support for most chains, bridges, and DEX aggregators as well as single DEXs. List of DEXs they support can be found [here](https://docs.li.fi/list-chains-bridges-dexs). 

* [LiFi Widget](https://docs.li.fi/integrate-li.fi-widget/li.fi-widget-overview) has a set of Prebuilt UI components that will help integrate cross-chain bridging and swapping experince. 

* [LiFi APIs](https://docs.li.fi/li.fi-api/li.fi-api) can be used to transfer tokens, request supported chains and tokens, token information, and all possible connections. You can also request status of transactions via the API. 

* [LiFi SDK](https://docs.li.fi/integrate-li.fi-js-sdk/install-li.fi-sdk) package allows access to Li.Fi API, and find the best cross chain routes on different bridges and exchanges. The routes can then be executed via the SDK.

- **Integration and Functionality Questions**:
  - How will the bridge talk to the smart contract?
  - Noting the challenge that the li.fi API/SDK uses web 2 to get quotes and confirm transactions.
  - Chainlink Automation triggers smart contract functions.
  - Consideration: Maybe use Chainlink Functions (a different CL product) for off-chain logic and interacting with Li.Fi?

- **Li.Fi Testnet Deployments**:
  - Ethereum Goerli
  - Polygon Mumbai

## Design considerations
~~#### 1. Bridging Solution A using CL Functions and Li.Fi API~~
- User deposits asset A to an ERC4626 Vault
- Every 24 hours a Chainlink Function interacts with the LiFi API to RFQ a quote to bridge and swap
    - (this may require a chainlink automation keeper to trigger the call to the Function)
-  The data from quote is passed to a `bridge assets()` function on the vault smart contract as argument and bridge and swap is executed
-  On the destination chain a chainlink keeper watches for the new asset and grabs the execution data for the swap - slippage etc
-  This execution data is passed back to the home chain vault by CCIP
-  This data is then used by a `updateAssets()` function that updates the accouting on the vault

#### 2. Bridging Solution B using CCIP
- Similar to the above but must use `CCIP-BnM` test tokens as deposit asset in vault and requires us to deploy a UNI V2 Pool on Sepolia that pairs the `CCIP-BnM` token against a ERC20 that we deploy
- User deposits token to Vault
- Every 24 hours a CL keeper triggers a bridge and swap using CCIP
- Call data for the swap is sent with the asset
- Swap is executed on Uni pool

~~#### 3. Custody Solution A using a EOA or very simple smart contract~~
- We basically build the bare minimum required to hold the asset and allow the swap once it has been bridged
- Pro: Easier to build and test, faster to deploy
- Con: One way trip for users. They are never getting money back...

#### 4. Custody Solution B using another vault
- We deploy another 4626 on the destination chain that accepts the bridged assets
- When a user wishes to withdraw their money, they burn their shares on the home chain and this creates a CCIP message to the destination chain vault to mint shares for them to withdraw.
- Pro: User can redeem their assets
- Con: Harder to build, more places for accounting to mess up

#### 5. Locking & Security
- As an additional precaution we can create a way to lock the vault when the bridge and swap is being executed
- For Example:
    - We lock the home chain vault (no deposits, withdrawals, mint, redeems etc) with the same CL keeper command that executes the bridge and swap
    - Once the swap has been executed on the destination chain, we can send a call back to the source chain to unlock the vault
