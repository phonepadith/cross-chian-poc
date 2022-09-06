# Deploying a EVM->EVM Cross Chain Token Bridge
## Introduction
This tutorial (Proof of Concept) walks through the process of deploying a token exchange bridge between two Ethereum test networks (Rinkeby and Ropsten). It could similarly be applied to implement any blockchain with two EVM-based chains including the Ethereum mainnet or BSC.

This tutorial is modification from this original: https://chainbridge.chainsafe.io/live-evm-bridge/  

## Preparing 
Before begin, we need to install all of softwares are shown below: 
- OS: Ubuntu 20.04 LTS
- Nodjs: v12.22.10
- Go: go1.16.7 linux/amd64
- gcc build-essential

## Concepts
At a high level setting up this workshop for token transfers between two chains requires the following:
- A token native to ChainA: **Rinkeby** (e.g. COS) and a destination token on ChainB: **Ropsten** which will represent the source token wrapped (e.g. wCOS).
- Create contracts and deployed for each chain. 
- Handler contracts on each chain to instruct the bridge what to do with the tokens One or more off-chain relayers running to pick up transactions from the source chain and relay them to the destination chain.

## Overview of Cross-chain Token Transfers
Tokens are inherently native to a single chain; however, with a bit of contract magic it is possible to make something equivalent to transferring value between two chains.
### Lock-and-Mint | Burn-and-Release
This approach is appealing because it imposes very few requirements on the asset on the source chain other than it must be transferable and lockable in a contract. This makes it possible to use with native assets (e.g. ETH) or existing tokens that you don't control.

![Overview Transaction](/img/Test-Work.png)

The basic flow of lock-and-mint is as follows:

1. Assets on the source chain are deposited in a bridge contract which locks them up.
2. A relayer observes this transaction and sends a new transaction to the bridge contract on the. destination chain. This transaction should include a proof of the deposit on the source chain and a destination which is owned by the depositor.
3. The bridge contract mints new tokens on the destination chain into the depositors account on this chain.

It is important to notice that the total number of liquid (non-locked) tokens on both chains combined remains the same. Exchanging the tokens on the destination chain back to native tokens uses the inverse operation, burn-and-release.

1. Tokens on the destination chain are sent to a bridge contract which burns them.
2. A relayer observes this transaction and sends a new transaction to the bridge contract on the source chain. This transaction should include a proof of the burn and a destination address.
3. The bridge contract unlocks some number of tokens and deposits them into the destination account.

Provided this refunding of tokens can be executed at any time and the number of locked tokens is always equal to the number of minted wrapped tokens, then we can say that the wrapped tokens have value equal to the original asset

## ChainBridge Components

A ChainBridge deployment on EVM-based chains requires the following components:

### Bridge Contract
The bridge contract must be deployed on both the source and destination chains. Its primary task on the source chain is to broker token deposits (ensure they are locked) and emit the events that the relayers listen for. On the destination chain, the bridge contract is responsible for managing a set of permissioned relayers, aggregating relayer votes on proposals passed from the source chain, and executing the desired action (e.g. minting tokens) on the destination chain when the vote threshold is reached.

### Handlers
To allow extensibility the bridge contract is written to call functions in a handler contract when tokens are deposited on the source chain or when a proposal is approved on the destination chain. Each resource (e.g. token) can have a unique handler. You are free to write your own handler contract that implements any functionality you like.

The ERC20 handler contract that ships with ChainBridge can be configured to either lock up the tokens or burn them (if the token allows) on deposit and either mint or release tokens when a proposal is executed. A single handler can handle many tokens but their contract addresses must be registered with the handler ahead of time.

### Relayers
The relayer is an off-chain actor that listens for particular events on the source chain and- when certain conditions are met- will submit signed proposals to the destination chain. The addresses of the approved relayers must be registered with the bridge contract on the destination chain.

Once a proposal has sufficient votes a relayer can execute the proposal to trigger the handler.

## Deploying your own bridge
### Preparation
***Accounts***
If you want to follow along with this guide we will be deploying a bridge between two Ethereum test networks (Rinkeby and Ropsten).

You will need one account on each network from which to deploy the contracts. These can be easily created using MetaMask. Be careful to use test accounts only as some of the commands in this tutorial require access to your private key.

This will cost gas so some test ETH will be required. So first up grab some test ether from the faucets:

**Rinkeby**: 
https://rinkebyfaucet.com/

**Ropsten**: 
https://faucet.metamask.io/


You will need around 0.1 each of Rinkeby ETH and Ropsten ETH.

We will be creating a bridge that wraps the test ERC20 token **COS** on Rinkeby as a wrapped version (**wCOS**) on Ropsten. So also grab some free COS tokens by sending a 0 ETH transaction to the contract address on Rinkeby: 0x6E7C2D85B4f94BBa02fa28845DF2cC27645E0087

***Create Token COS***
In this case we use those contracts to deploy COS token. This token can mint/burn also:

***IERC20.sol***
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

import "./IERC20.sol";

contract ERC20 is IERC20 {
    uint public totalSupply;
    mapping(address => uint) public balanceOf;
    mapping(address => mapping(address => uint)) public allowance;
    string public name = "CHAIN-CORSS";
    string public symbol = "COS";
    uint8 public decimals = 18;

    function transfer(address recipient, uint amount) external returns (bool) {
        balanceOf[msg.sender] -= amount;
        balanceOf[recipient] += amount;
        emit Transfer(msg.sender, recipient, amount);
        return true;
    }

    function approve(address spender, uint amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }

    function transferFrom(
        address sender,
        address recipient,
        uint amount
    ) external returns (bool) {
        allowance[sender][msg.sender] -= amount;
        balanceOf[sender] -= amount;
        balanceOf[recipient] += amount;
        emit Transfer(sender, recipient, amount);
        return true;
    }

    function mint(uint amount) external {
        balanceOf[msg.sender] += amount;
        totalSupply += amount;
        emit Transfer(address(0), msg.sender, amount);
    }

    function burn(uint amount) external {
        balanceOf[msg.sender] -= amount;
        totalSupply -= amount;
        emit Transfer(msg.sender, address(0), amount);
    }
}

```
***ERC20.sol***
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

// https://github.com/OpenZeppelin/openzeppelin-contracts/blob/v3.0.0/contracts/token/ERC20/IERC20.sol
interface IERC20 {
    function totalSupply() external view returns (uint);

    function balanceOf(address account) external view returns (uint);

    function transfer(address recipient, uint amount) external returns (bool);

    function allowance(address owner, address spender) external view returns (uint);

    function approve(address spender, uint amount) external returns (bool);

    function transferFrom(
        address sender,
        address recipient,
        uint amount
    ) external returns (bool);

    event Transfer(address indexed from, address indexed to, uint value);
    event Approval(address indexed owner, address indexed spender, uint value);
}
```