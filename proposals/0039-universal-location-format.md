---
Title: Standardizing Universal Location Format (ULF) for Cross-Blockchain Asset Reference
Number: 0
Status: Draft
Version: 0.1
Authors:
 - LAOS parachain ( https://www.laosfoundation.io/ )
Created: 2024-01-29
Impact: Low
Requires: NA
Replaces: NA
---

## Summary

The proposal seeks to establish a standardized Universal Location Format (ULF) across varied blockchain ecosystems, leveraging the existing solution of identifying resources within consensus systems as utilized in Cross-Consensus Messaging (XCM). The ULF is essentially a stringified adaptation of the universal location concept employed in XCM, ensuring a consistent and unambiguous approach to asset referencing. This standardization aims to provide decentralized applications (dApps) with a common methodology to reference assets, particularly NFTs, across a wide range of blockchains such as Ethereum, Polkadot, Kusama, among others. 

By aligning the ULF closely with the well-established framework of XCM, the proposal intends to streamline the process of asset identification, retrieval, and interaction, thereby offering a uniform, clear, and efficient string format for referencing assets within and across different blockchain systems.

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

- Ethereum NFT: `uloc://GlobalConsensus(7)/AccountKey20(0xabc....def)/GeneralKey(1234)`
- EVM Parachain NFT in Kusama: `uloc://GlobalConsensus(3)/Parachain(2023)/PalletInstance(51)/AccountKey20(0xabc....def)/GeneralKey(4321)`

## Security Considerations

When considering the security implications of the Universal Location Format (ULF), it is crucial to emphasize the importance of case sensitivity and strict formatting standards. These aspects are vitally important for maintaining consistency and ensuring security, especially in operations involving cryptographic functions like hashing.

1. **Case Sensitivity and String Strictness:**
   - The ULF must be defined with clear rules regarding case sensitivity. Different blockchain platforms may have varying standards for address case sensitivity. However, for the ULF, a consistent approach must be established and adhered to across all platforms.
   - When a ULF is stored on a blockchain, it may be hashed for various purposes, including integrity checks or as a part of cryptographic operations. Any inconsistency in case handling can lead to different hashes for the same ULF, leading to discrepancies and potential security vulnerabilities.
   - A strict and well-defined format for the ULF ensures that all participants in the system interpret and use the format consistently, reducing the risk of errors or inconsistencies that could be exploited.

## Impact

The introduction of the Universal Location Format (ULF) brings significant advancements in the way decentralized applications (dApps) and smart contracts interact with and reference assets across multiple blockchain ecosystems. Specifically:

1. **Enhanced Interoperability for ERC721 Tokens:**
   - By incorporating the ULF into the `tokenURI` of ERC721 tokens, we can significantly enhance the interoperability of Non-Fungible Tokens (NFTs). The ULF provides a standardized, cross-chain referencing mechanism, enabling NFTs to be more easily identified, managed, and utilized across different blockchain platforms.

2. **Streamlined Asset Management:**
   - The ULF simplifies the process of asset referencing by providing a uniform, unambiguous format. This reduces the complexity currently associated with managing and interacting with assets that exist on or interact with multiple blockchains, thereby streamlining operations for developers and users alike.

3. **Future-Proofing Asset References:**
   - The adoption of ULF as a standard ensures that asset references are future-proof. As the blockchain space continues to evolve and new platforms emerge, the ULF's structured and systematic approach to asset referencing can readily adapt to future developments, maintaining its relevance and utility.

In conclusion, the ULF's integration, especially within the ERC721 `tokenURI` standard, signifies an enhancement in asset interoperability and management across the ever-evolving landscape of blockchain technology.

## Alternatives

Alternatives such as continuing with blockchain-specific addressing were considered. However, they lack the universality and interoperability that ULF offers. While ULF adoption requires initial effort, the lack of a universal standard perpetuates inefficiencies and hinders the blockchain ecosystem's growth.

## Questions and Open Discussions

This section invites the community's input, insights, and active engagement to refine and optimize the implementation of the Universal Location Format (ULF). We encourage contributors to consider the following questions and to propose discussions on any related topics:

1. **Integration with Existing Systems:**
   - How can the ULF be seamlessly integrated into existing blockchain ecosystems and smart contract standards, such as ERC721's `tokenURI`, without causing disruptions or requiring extensive overhauls?

2. **Case Sensitivity and Standardization:**
   - Given the importance of case sensitivity and strict formatting in the ULF, what are the best practices for ensuring consistency across different blockchain platforms? How can we enforce these standards effectively?

3. **Handling Edge Cases:**
   - Are there any edge cases or unique scenarios in cross-blockchain asset referencing that the ULF might need to address? How can the ULF be designed to accommodate these situations?

4. **Community Feedback on Structure and Usability:**
   - What feedback does the community have regarding the structure, usability, and practicality of the ULF in real-world applications? Are there any suggestions for improving its design or functionality?

5. **Future-Proofing the ULF:**
   - As blockchain technology continues to evolve, how can the ULF adapt to support new platforms, protocols, and consensus mechanisms that may emerge in the future?

We welcome diverse perspectives and constructive discussions to ensure that the ULF not only meets the current demands of asset referencing across blockchains but also remains robust, flexible, and relevant in the face of future technological advancements.

## References

"Universal Location Format Across Blockchains Discussion", freeverseio/laos, (2021). Retrieved from [GitHub issue #177](https://github.com/freeverseio/laos/issues/177).

"Cross Consensus Messaging (XCM) - Data Structures and Interfaces", paritytech/xcm, (2021). Retrieved from [GitHub repository](https://github.com/paritytech/xcm).

"Uniform Resource Identifier (URI): Generic Syntax", T. Berners-Lee, R. Fielding, L. Masinter, (2005). Retrieved from [RFC 3986](https://tools.ietf.org/html/rfc3986).
