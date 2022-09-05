# Deploying a EVM->EVM Cross Chain Token Bridge
## Introduction
This tutorial (Proof of Concept) walks through the process of deploying a token exchange bridge between two Ethereum test networks (Rinkeby and Ropsten). It could similarly be applied to implement any blockchain with two EVM-based chains including the Ethereum mainnet or BSC.

This tutorial is modification from: https://chainbridge.chainsafe.io/live-evm-bridge/  

## Preparing 
Before begin, we need to install all of softwares are shown below: 
- OS: Ubuntu 20.04 LTS
- Nodjs: v12.22.10
- Go: go1.16.7 linux/amd64
- gcc build-essential

## Concepts
At a high level setting up this workshop for token transfers between two chains requires the following:
- A token native to ChainA: Rinkeby (e.g. COS) and a destination token on ChainB: Ropsten which will represent the source token wrapped (e.g. wCOS).
- Create contracts and deployed for each chain. 
- Handler contracts on each chain to instruct the bridge what to do with the tokens One or more off-chain relayers running to pick up transactions from the source chain and relay them to the destination chain.

## Overview of Cross-chain Token Transfers