---
title: "Native EVM Contracts on Polkadot"
description: "A Summary for Native EVM Contracts on Polkadot"
date: 2025-05-12
image: "/images/posts/smart-contract-illustration.png"
categories: ["Smart Contract"]
authors: ["Husni"]
tags: ["EVM", "Smart Contract"]
draft: false
---

This is a presentation about PolkaVM that I made to present to one of local web3 communities. Since it's currently in development, things might have changed and what you're going to read might not be valid anymore. Feel free to consult the official docs.

## Overview

- Execute EVM contracts natively on Substrate-based chains
- Uses `pallet-revive` in Substrate runtime
- Supports Ethereum-like transactions and state transitions
- Benefits: Lower gas fees, faster transactions, interoperability with Substrate

---

## What is PolkaVM?

PolkaVM is a cutting-edge virtual machine tailored to **optimize smart contract execution** on Polkadot. 

Unlike traditional Ethereum Virtual Machines (EVM), PolkaVM is built with a **RISC-V-based register architecture** and a **64-bit word size**.

---

## Why RISC-V?

- Open Standard: allows for greater customization and innovation in VM design.
- Versatile: easy to transpile into common hardware formats such as x86, x64, and ARM.
- Simplicity and efficiency: simple and clean ISA leads to more efficient and easier optimization of smart contracts.
- 100x efficiency gains for ZK-EVM, see Vitalik proposal [here](https://ethereum-magicians.org/t/long-term-l1-execution-layer-proposal-replace-the-evm-with-risc-v/23617).

---

## PolkaVM vs EVM (Part 1)

| Feature | PolkaVM | EVM |
|---------|---------|-----|
| Architecture | RISC-V-based register | Stack-based |
| Word Size | 64-bit | 256-bit |
| Optimization | Tailored for smart contract execution on Polkadot | General-purpose for Ethereum |
| Arithmetic Operations | Faster due to register architecture | Slower due to stack operations |
| Architecture | RISC-V-based register | Stack-based |
| Word Size | 64-bit | 256-bit |


---

## PolkaVM vs EVM (Part 2)


| Feature | PolkaVM | EVM |
|---------|---------|-----|
| Hardware Translation | More efficient | Less efficient |
| Language Integration | Seamless with C and Rust | Primarily Solidity |
| Scalability | Improved for modern blockchain applications | Limited by design |
| Ecosystem | Newer, growing | Well-established |


--- 


## PolkaVM vs EVM (Part 3)

| Feature | PolkaVM | EVM |
|---------|---------|-----|
| Gas Efficiency | Potentially more efficient with precise metering | Less efficient for complex operations |
| Memory Management | More flexible with 64-bit addressing | Limited by 256-bit word size |


---

## PolkaVM - Key Benefits

- Faster arithmetic operations and efficient hardware translation
- Seamless integration of high-performance languages like C and Rust for advanced optimization
- Improved scalability for modern blockchain applications

---

## Key Components

- **pallet_revive**: Runtime module executing smart contracts, converting Ethereum transactions for blockchain compatibility

- **PolkaVM**: Custom RISC-V-based VM optimized for performance, supporting Solidity and high-performance languages

- **Revive**: Compiler translating Solidity to RISC-V for PolkaVM, ensuring full Solidity compatibility

- **Revive Remix**: Modified Remix IDE with LLVM-based Revive backend for efficient Solidity contract deployment on Polkadot

---

## Key Considerations

- Gas Pricing: Custom or Ethereum model
- Account Management: Handle both Substrate and Ethereum accounts
- Storage: Manage EVM state in Substrate storage
- Performance: Balance EVM execution and chain performance

---

## Development Tools

- Remix: Web-based Solidity IDE
- Hardhat: Development environment for Ethereum
- Truffle: Testing framework for Ethereum
- Web3.js: JavaScript library for Ethereum interaction
- Ethers.js: Alternative JavaScript library for Ethereum

---

## DOOM Demo!

https://youtu.be/hJcw5FMSjQs?si=WSOk0prFh6hVo9oH

---

## Key Takeaway

You can run normal software compiled from normal programming languages (Rust, C, C++, etc) using normal abstractions on Polkadot _soon_.


--- 

## References

- [Polkadot Official Docs](https://docs.polkadot.com/develop/smart-contracts/evm/native-evm-contracts/)
- [Official Docs for Asset Hub smart contracts](https://contracts.polkadot.io/)
- [PolkaVM](https://github.com/paritytech/polkavm)
- [DOOM Port for PolkaVM](https://github.com/koute/polkadoom)
- [Asset Hub Block Explorer](https://blockscout-asset-hub.parity-chains-scw.parity.io)
