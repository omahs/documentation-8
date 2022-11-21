---
layout: nodes.liquid
section: ethereum
date: Last Modified
title: 'Direct Funding Method'
permalink: 'docs/vrf/v2/direct-funding/'
whatsnext:
  {
    'Get a Random Number': '/docs/vrf/v2/direct-funding/examples/get-a-random-number/',
    'Supported Networks': '/docs/vrf/v2/direct-funding/supported-networks/',
  }
metadata:
  title: 'Generate Random Numbers for Smart Contracts using Chainlink VRF v2 - Direct funding method'
  description: 'Learn how to securely generate random numbers for your smart contract with Chainlink VRF v2. This guide uses the Direct funding method.'
---

{% include 'sections/vrf-v2-directfunding-common.md' %}

This guide explains how to generate random numbers using the Direct funding method. This method doesn't require a subscription and is optimal for one-off requests for randomness. This method also works best for applications where your end-users must pay the fees for VRF because the cost of the request is determined at request time.

**Topics**

- [VRF Direct funding](#vrf-direct-funding)
- [Request and Receive Data](#request-and-receive-data)
  - [End To End Diagram](#end-to-end-diagram)
  - [Explanation](#explanation)
- [Limits](#limits)

## VRF Direct funding

Unlike the [subscription method](/docs/vrf/v2/subscription/), the Direct funding method does not require you to create subscriptions and pre-fund them. Instead, you must directly fund consuming contracts with LINK tokens before they request randomness.

For Chainlink VRF v2 to fulfill your requests, you must have a sufficient amount of LINK in your consuming contract. Gas cost calculation includes the following variables:

- **Gas price:** The current gas price, which fluctuates depending on network conditions.

- **Callback gas:** The amount of gas used for the callback request that returns your requested random values.

- **Verification gas:** The amount of gas used to verify randomness on-chain.

- **Wrapper overhead gas:** The amount of gas used by the VRF Wrapper contract. See the [Request and Receive Data](#request-and-receive-data) section for details about the VRF v2 Wrapper contract design.

The gas price depends on current network conditions. The callback gas depends on your callback function and the number of random values in your request. You define the limits that you are willing to spend for the request with the following variable:

- **Callback gas limit:** Specifies the maximum amount of gas you are willing to spend on the callback request. Define this limit by specifying the `callbackGasLimit` value in your request.

> ℹ️ Note on transaction costs.
>
> Because the consuming contract directly pays the LINK for the request, the cost is calculated during the request and not during the callback when the randomness is fulfilled. Test your callback function to learn how to correctly estimate the callback gas limit.
>
> - If the gas limit is underestimated, the callback fails and the consuming contract is still charged for the work done to generate the requested random values.
> - If the gas limit is overestimated, the callback function will be executed but your contract is not refunded for the excess gas amount that you paid.
>
> Make sure that your consuming contracts are funded with enough LINK tokens to cover the transaction costs. If the consuming contract doesn't have enough LINK tokens, your request will revert.

## Request and Receive Data

### End To End Diagram

![Vrf v2 Direct funding method end to end diagram](/images/vrf/v2-direct-funding-e2e.webp)

Two types of accounts exist in the Ethereum ecosystem:

- EOA (Externally Owned Account): An externally owned account that has a private key and can control a smart contract. Transactions can be initiated only by EOAs.
- Smart contract: A smart contract that does not have a private key and executes what it has been designed for as a decentralized application.

The Chainlink VRF v2 solution uses both off-chain and on-chain components:

- [VRF v2 Wrapper (on-chain component)](https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/VRFV2Wrapper.sol): A wrapper for the VRF Coordinator that provides an interface for consuming contracts.
- [VRF v2 Coordinator (on-chain component)](https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/VRFCoordinatorV2.sol): A contract designed to interact with the VRF service. It emits an event when a request for randomness is made, and then verifies the random number and proof of how it was generated by the VRF service.
- VRF service (off-chain component): Listens for requests by subscribing to the VRF Coordinator event logs and calculates a random number based on the block hash and nonce. The VRF service then sends a transaction to the `VRFCoordinator` including the random number and a proof of how it was generated.

### Explanation

Requests to Chainlink VRF v2 follow the [Request & Receive Data](#request-and-receive-data) cycle. The VRF wrapper calls the coordinator to process the request using the following steps:

1. The consuming contract must inherit [VRFV2WrapperConsumerBase](https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/VRFV2WrapperConsumerBase.sol) and implement the `fulfillRandomWords` function, which is the _callback VRF function_. Submit your VRF request by calling the `requestRandomness` function in the [VRFV2WrapperConsumerBase](https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/VRFV2WrapperConsumerBase.sol) contract. Include the following parameters in your request:

   - `requestConfirmations`: The number of block confirmations the VRF service will wait to respond. The minimum and maximum confirmations for your network can be found [here](/docs/vrf/v2/direct-funding/supported-networks/#configurations).
   - `callbackGasLimit`: The maximum amount of gas to pay for completing the callback VRF function.
   - `numWords`: The number of random numbers to request. You can find the maximum number of random values per request for your network in the [Supported networks](/docs/vrf/v2/direct-funding/supported-networks/#configurations) page.

1. The consuming contract calls the [VRFV2Wrapper](https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/VRFV2Wrapper.sol) `calculateRequestPrice` function to estimate the total transaction cost to fulfill randomness. Then the consuming contract calls the [LinkToken](https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.4/LinkToken.sol) `transferAndCall` function to pay the wrapper with the calculated request price. This method sends LINK tokens and executes the [VRFV2Wrapper](https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/VRFV2Wrapper.sol) `onTokenTransfer` logic. This triggers the VRF [VRF Coordinator](https://github.com/smartcontractkit/chainlink/blob/develop/contracts/src/v0.8/VRFCoordinatorV2.sol) `requestRandomWords` function to request randomness.
   The final gas cost to fulfill randomness is estimated based on how much gas is expected for the verification and callback. The total gas cost in wei uses the following formula:

   ```
   (Gas price * (Verification gas + Callback gas limit + Wrapper gas Overhead)) = total gas cost
   ```

   The total gas cost is converted to LINK using the ETH/LINK data feed. In the unlikely event that the data feed is unavailable, the VRF Wrapper uses the `fallbackWeiPerUnitLink` value for the conversion instead. The `fallbackWeiPerUnitLink` value is defined in the [VRF v2 Wrapper contract](/docs/vrf/v2/direct-funding/supported-networks/#configurations) for your selected network.

   A LINK premium is then added to the total gas cost. The premium is divided in two parts:

   - Wrapper premium: The premium percentage. You can find the percentage for your network in the [Supported networks](/docs/vrf/v2/direct-funding/supported-networks/#configurations) page.
   - Coordinator premium: A flat fee. This premium is defined in the `fulfillmentFlatFeeLinkPPMTier1` parameter in millionths of LINK. You can find the flat fee of the coordinator for your network in the [Supported networks](/docs/vrf/v2/direct-funding/supported-networks/#configurations) page.

   ```
   ((total gas cost * Wrapper premium) + Coordinator premium) = total request cost
   ```

1. The VRF coordinator emits an event.

1. The event is picked up by the VRF service and waits for the specified number of block confirmations to respond back to the VRF coordinator with the random values and a proof (`requestConfirmations`).

1. The VRF coordinator verifies the proof on-chain. Then, it calls back the wrapper contract `fulfillRandomWords` function.

1. Finally, the VRF Wrapper calls back your consuming contract.

## Limits

You can see the configuration for each network on the [Supported networks](/docs/vrf/v2/direct-funding/supported-networks/) page. You can also view the full configuration for each VRF v2 Wrapper contract directly in Etherscan. As an example, view the [Ethereum Mainnet VRF v2 Wrapper contract](https://etherscan.io/address/0x5A861794B927983406fCE1D062e00b9368d97Df6#readContract) configuration by calling `getConfig` function.

- Each wrapper has a `maxNumWords` parameter that limits the maximum number of random values you can receive in each request.

> ℹ️ Note on maximum gas limit.
>
> The maximum allowed `callbackGasLimit` value for your requests is defined in the [Coordinator contract supported networks](/docs/vrf/v2/subscription/supported-networks/) page. Because the VRF v2 Wrapper adds an overhead, your `callbackGasLimit` must not exceed `maxGasLimit - wrapperGasOverhead`.