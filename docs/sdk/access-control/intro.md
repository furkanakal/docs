---
sidebar_position: 1
---

# Intro

## Quick Start

Ready to jump right in? Quickly learn how you can integrate decentralized access control into your own product:

1. Guide: [Working with Decentralized Access Control](https://spark.litprotocol.com/working-with-decentralized-access-control/)
2. Guide: [Encrypting and Decrypting Content with Lit](../access-control/encryption.md)
3. Tool: [Custom Access Controls Creator](https://custom-access-control-conditions.lit.repl.co/) 
4. Example: [Basic EVM Conditions](../access-control/evm/basic-examples)

## Overview

Lit Protocol provides developers with a decentralized access control layer that can be used to [encrypt](../../resources/glossary#encryption) content for private and permissioned storage on the open Web. The [Lit SDK](https://github.com/LIT-Protocol/js-sdk) provides utilities that can be used for encrypting and decrypting content client-side, while [access control conditions](../access-control/condition-types/unified-access-control-conditions) (ACCs) are used to define who can decrypt and access the locked data. 

Lit supports the use of both on and [off-chain data](../access-control/condition-types/lit-action-conditions) when defining access control conditions. Examples include gating against:

- [Membership within a particular DAO](../access-control/evm/basic-examples#must-be-a-member-of-a-dao-molochdaov21-also-supports-daohaus)
- Ownership of a particular [ERC-721](../access-control/evm/basic-examples#must-posess-any-token-in-an-erc721-collection-nft-collection) or [ERC-20](../access-control/evm/basic-examples#must-posess-at-least-one-erc20-token) token
- The result of [any smart contract call](../access-control/evm/custom-contract-calls)
- The result of [any API call](../access-control/condition-types/lit-action-conditions), such as a follow on Twitter

## Features

1. Access Control Conditions are compatible with most EVM chains, Cosmos, and Solana. View the full list [here](../../resources/supported-chains.md).
2. AND + OR operators ([boolean logic](../access-control/condition-types/boolean-logic)) can be used to combine any of the supported conditions listed above.
3. Storage provider agnostic: use your preferred storage solution, including [IPFS](https://spark.litprotocol.com/encrypttoipfs/), Arweave, Ceramic, or even a centralized provider, like AWS.

## Examples and Use Cases

1. [Private data](https://docs.lens.xyz/docs/gated) for web3 social
2. [Token-gated video](https://github.com/suhailkakar/livepeer-token-gated-vod) streaming
3. [Encrypted token metadata](https://spark.litprotocol.com/semantic/)
4. [Persistent and private data marketplaces](https://blog.streamr.network/streamr-integrates-lit-protocol/)
5. Token-gating access to apps, [such as Streamlit](https://github.com/AlgoveraAI/streamlit-metamask/tree/main#lit-protocol-components)