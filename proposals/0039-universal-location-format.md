---
Title: Standardizing Universal Location Format (ULF) for Cross-Blockchain Asset Reference
Number: 0
Status: Draft
Version: 1.0
Authors:
 - [LAOS parachain core team](https://www.laosfoundation.io/)
Created: 2024-01-29
Impact: High
Requires: NA
Replaces: NA
---

## Summary

The proposal seeks to establish a standardized Universal Location Format (ULF) across varied blockchain ecosystems, leveraging the existing solution of identifying resources within consensus systems as utilized in Cross-Consensus Messaging (XCM). The ULF is essentially a stringified adaptation of the universal location concept employed in XCM, ensuring a consistent and unambiguous approach to asset referencing. This standardization aims to provide decentralized applications (dApps) with a common methodology to reference assets, particularly NFTs, across a wide range of blockchains such as Ethereum, Polkadot, Kusama, among others. By aligning the ULF closely with the well-established framework of XCM, the proposal intends to streamline the process of asset identification, retrieval, and interaction, thereby offering a uniform, clear, and efficient string format for referencing assets within and across different blockchain systems.

## Motivation

As the blockchain space expands, the interoperability between different blockchain ecosystems has become paramount. Currently, each blockchain has its unique addressing system, leading to confusion and inefficiencies when referencing assets across platforms. The Universal Location Format (ULF) addresses this issue by providing a standardized, cross-chain referencing system. This format not only facilitates smoother asset management and dApp development but also promotes a more interconnected and efficient blockchain ecosystem. Adopting ULF will benefit developers, users, and the broader blockchain community by reducing complexity and enhancing the interoperability of digital assets across various platforms.

## Specification

The Universal Location Format (ULF) is defined as:

``` 
uloc://GlobalConsensus(networkId)/Parachain(paraId)/PalletInstance(evmPalletId)/AccountKey20(smartContractAddress)/GeneralKey(tokenId)
```


- **GlobalConsensus(networkId):** Identifies the network consensus, with networkId indicating specific blockchain networks as defined in the xcm-format.
- **Parachain(paraId):** (Optional) Required only for assets located in Parachains within the Polkadot/Kusama ecosystem.
- **PalletInstance(evmPalletId):** Specifies the instance of the pallet where the asset is located.
- **AccountKey20(smartContractAddress):** Represents the address of the smart contract managing the asset.
- **GeneralKey(tokenId):** The unique identifier of the asset within the smart contract.

This format is in line with the XCMv3 format and adheres to the hierarchical nature of the MultiLocation structure used in the Polkadot ecosystem.

### Examples

- Ethereum NFT: `uloc://GlobalConsensus(7)/AccountKey20(0xabc....def)/GeneralKey(666)`
- EVM Parachain NFT in Kusama: `uloc://GlobalConsensus(3)/Parachain(2023)/PalletInstance(51)/AccountKey20(0xabc....def)/GeneralKey(666)`

## Security Considerations

Security is a paramount concern in the design of the ULF. The format is constructed to minimize ambiguity and prevent address collisions. However, it's essential to consider the following:

- The proper validation of ULF strings to prevent injection attacks or reference to malicious contracts.
- Ensuring that the ULF adheres to the security protocols of the individual blockchains it references.
- Regular audits and updates in response to newly discovered vulnerabilities in the blockchain space.

## Impact

Adopting the ULF involves changes in how dApps reference assets across blockchains. It necessitates updates in dApp development practices and possibly minor adjustments in existing smart contracts to support ULF. However, the long-term benefits in terms of interoperability and ease of asset management outweigh the initial adoption overhead.

## Alternatives

Alternatives such as continuing with blockchain-specific addressing were considered. However, they lack the universality and interoperability that ULF offers. While ULF adoption requires initial effort, the lack of a universal standard perpetuates inefficiencies and hinders the blockchain ecosystem's growth.

## Questions and Open Discussions (optional)

- Feedback on the ULF structure and its compatibility with various blockchain ecosystems.
- Suggestions for the seamless integration of ULF into existing blockchain infrastructures.
- Discussion on potential improvements or extensions to the ULF to cover more use cases and assets.

## Reference

Initial discussions and considerations regarding the Universal Location Format (ULF) can be found in the GitHub issue: [freeverseio/laos/issues/177](https://github.com/freeverseio/laos/issues/177). This issue laid the groundwork for the development and standardization of the ULF, addressing the need for a unified asset referencing system across different blockchain ecosystems.
