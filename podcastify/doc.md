# What is OP_CAT? How BIPs could bring OP_CAT back | OKX
The Bitcoin Improvement Proposal (BIP), introduced in 2011, allows the Bitcoin community to suggest, discuss, and implement changes to Bitcoin's protocol. Anyone can start a BIP by drafting a detailed proposal, which the community reviews. If the proposal is considered valuable, it's formally submitted for approval.

As you can imagine, the Bitcoin community is highly opinionated — if you're planning to write a BIP, you better make sure you're on point. In October 2023, Ethan Heilman and Armin Sabouri co-wrote and submitted a [BIP](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2023-October/022049.html?ref=hashrateindex.com) that made a case for enabling OP\_CAT, a Bitcoin scripting feature that Satoshi removed in 2010.

In April 2024, Heilman and Sabouri received ["BIP number" 347](https://www.coindesk.com/tech/2024/04/24/op-cat-proposal-to-bring-smart-contracts-to-bitcoin-finally-gets-a-bip-number), which isn't a seal of approval from the Bitcoin community, but the start of many debates and one step in a long journey to gain approval. In this article, we explore what OP-CAT is, what its reintroduction could mean for the Bitcoin network, and help you understand BIPs in general.

TL;DR
-----

*   The Bitcoin Improvement Proposal (BIP) process, launched in 2011, lets the community suggest and implement changes to Bitcoin's protocol.
    
*   Any community member can draft a BIP, which undergoes rigorous review and approval based on community feedback.
    
*   Proposing a BIP requires careful planning to navigate the diverse opinions within the Bitcoin community.
    
*   In October 2023, Ethan Heilman and Armin Sabouri submitted a BIP to improve Bitcoin's scripting by bringing back OP\_CAT.
    
*   Heilman and Sabouri's proposal gained the BIP number 347 in April 2024.
    

What is a BIP and the surrounding process?
------------------------------------------

The BIP process is a way for the community to suggest, discuss, and change Bitcoin's protocol. It's similar to how a city council might gather opinions from residents before changing local laws.

Inspired by [Python Enhancement Proposals](https://peps.python.org/) (PEPs), the [BIP](https://github.com/bitcoin/bips) process was introduced in 2011 by Amir Taaki, a Bitcoin developer. The proposal establishes a structured process for analyzing suggested changes to the Bitcoin protocol, making sure that all voices within the community are heard and considered.

Here's how the BIP process works:

### Proposal initiation

The process begins with an idea, which could come from any community member. This idea is informally discussed in forums, including platforms like [Bitcoin Talk](https://bitcointalk.org/) and [X](http://x.com/).

### Drafting

If the idea gains traction, the proposer writes a detailed draft explaining the proposed change. This draft, or BIP, outlines the technical details, rationale, and potential impact on the Bitcoin network.

### Community review

The draft is shared with the community to get feedback. It's similar to proposing a new feature in a software update, where developers and users give their opinions and suggestions. Revisions are often made based on this feedback.

### Formal submission

After review, the BIP is submitted for approval. If it's a technical change, miners and node operators may show support by marking the blocks they mine.

### Activation

When there's wide enough agreement, the BIP can be put into action. Different methods, like the "[Speedy Trial](https://bitcoinmagazine.com/technical/analyzing-bip119-and-the-controversy-surrounding-it)" mechanism used for the Taproot Bitcoin upgrade, can be applied.

Overall, the BIP process makes sure that Bitcoin protocol changes are made democratically and transparently, creating a decentralized development environment. An inclusive approach maintains Bitcoin's integrity and adaptability, making sure the network evolves with the community's needs and the overall consensus.

If you're curious about BIPs, you can visit the [Bitcoin BIPs repository](https://github.com/bitcoin/bips) to stay updated on current discussions and proposals.

### What is OP\_CAT?

OP\_CAT is a Bitcoin feature that expands its scripting abilities. It combines data into a single output — a process in programming known as concatenation — making transactions simpler and allowing for the development of complex smart contracts.

When Satoshi Nakamoto first introduced OP\_CAT, it was removed due to potential abuse. The rationale was that too much data on the blockchain may cause a denial of service (DoS) attack.

### Concatenation explained

OP\_CAT applies concatenation by combining several pieces of transaction data into one report. Similar to merging puzzle pieces, the process simplifies complex transactions by linking data together. This allows for more advanced and interconnected operations in the Bitcoin ecosystem.

### Advanced scripting

With OP\_CAT, developers can create complex transactions with advanced scripting, which was previously a real challenge on the Bitcoin network. For example, OP\_CAT can be used to build sophisticated payment structures or conditional transactions that respond to specific conditions.

### Versatility

OP\_CAT is gaining attention for its potential to enhance Bitcoin's capabilities. It's part of a trend to make Bitcoin a more versatile platform for complex applications. Continuous enhancements are being tested to expand Bitcoin's use cases, such as [Runes](https://www.okx.com/learn/runes-protocol) and [ordinals](https://www.okx.com/learn/ordinals).

### Challenges

The Bitcoin community is still debating the technical implications of reintroducing OP\_CAT. Some argue that it could complicate Bitcoin's codebase and introduce security risks, while others believe its benefits outweigh the potential drawbacks. This debate questions how to balance simplicity and security with innovation in Bitcoin.

OP\_CAT's ability to concatenate data opens up new possibilities for Bitcoin applications, making it a hot topic in the ongoing dialogue about Bitcoin enhancements.

### What are the proposed use cases for OP\_CAT?

The reintroduction of OP\_CAT in [BIP 347](https://github.com/bitcoin/bips/pull/1525) could bring advanced functionalities, enhance Bitcoin smart contracts, and improve transaction security.

Here's a closer look at proposed OP\_CAT use cases.

### Bitcoin covenants

Bitcoin covenants use OP\_CAT to create specific conditions for spending Bitcoin. A legal trust limits how funds can be used. Similarly, covenants can restrict how Bitcoin is spent, making sure it only goes to a specified address or is used in specific ways. This adds an important layer of security for large holdings.

### Bitcoin vaults

Another use for OP\_CAT is in setting up Bitcoin vaults. Think of vaults as secure containers for Bitcoin that require multiple approvals or steps to open. For example, a vault might require confirmation over a period of time before funds can be spent, similar to a time-locked safe. This setup can protect against unauthorized transactions by adding a delay mechanism, making it difficult for an attacker to steal funds quickly.

### Non-equivocation contracts

Bitcoin payment channels and non-equivocation contracts can prevent double spending. It's like having a system that stops you from cashing the same check twice. If you try to spend the same Bitcoin on several payment channels, the contract recognizes it and enforces penalties. This helps to keep the transaction secure.

### Tree signatures

Tree signatures improve multisignature transactions that require multiple approvals, like in a corporate account. They organize signatures efficiently, reducing the data needed per transaction. This simplifies management and data usage, even for complex setups with many participants.

What are the challenges surrounding OP\_CAT's reintroduction?
-------------------------------------------------------------

The reintroduction of OP\_CAT is sparking heated debates within the Bitcoin community. Some believe it could boost Bitcoin's significance, while others worry it could weaken its simplicity, which is a core strength.

### Challenges and controversies

Critics say adding OP\_CAT could make the code harder to manage and increase the chance of problems arising. This concern forms a significant part of the [Bitcoin upgrade debate](https://www.bloomberg.com/news/articles/2024-06-01/bitcoin-debate-heats-up-over-software-revamp-to-add-new-features?embedded-checkout=true).

Consensus-building for changes like OP\_CAT is hard in the Bitcoin community. Agreement requires addressing different views and getting strong technical and community support. This involves detailed discussions on benefits, risks, and activation methods.

### The Bitcoin Community Debate

OP\_CAT could add new features to Bitcoin, making it more appealing and competitive with other cryptocurrencies like Ethereum, which already support complex smart contracts. However, opponents believe that such functionalities shouldn't come at the expense of Bitcoin's core principles of security and simplicity. This tension is central to the ongoing Bitcoin functionality vs. simplicity debate.

### OP\_CAT activation methods

There's a lot of debate in the community around OP\_CAT activation methods. The options are either a soft fork, which introduces changes in a backward-compatible manner, or a hard fork, which has the potential to split the network.

Both methods significantly affect network consensus and stability, making the choice highly contentious. The community must navigate these choices carefully to avoid fracturing consensus and make sure of a smooth transition.

The Bitcoin community's debate over OP\_CAT highlights the tension between enhancing Bitcoin's functionality and preserving its simplicity. As discussions continue, the community must carefully weigh the potential benefits against the risks to preserve the network's integrity and utility.

How does OP\_CAT compare to other Bitcoin enhancements?
-------------------------------------------------------

The potential return of OP\_CAT has sparked interest and comparisons with other Bitcoin enhancements. To understand its place in the evolving ecosystem, it's helpful to examine how OP\_CAT stands out from other protocols like OP\_CTV and ordinals.

### OP\_CTV vs OP\_CAT

Both OP\_CAT and [OP\_CTV (CheckTemplateVerify)](https://bitcoinops.org/en/topics/op_checktemplateverify) support Bitcoin's scripting capabilities, but they serve different purposes. OP\_CTV focuses on covenants, which are like rules for Bitcoin transactions. These covenants make sure that funds follow certain conditions.

OP\_CAT's unique functionality allows for the direct concatenation of data. This flexibility enhances transaction designs.

### Ordinals protocol

The ordinals protocol allows for the creation and transfer of NFTs (non-fungible tokens) on the Bitcoin blockchain. Unlike OP\_CAT, which extends Bitcoin's scripting capabilities for transactions, ordinals primarily focuses on the representation and transfer of assets.

Ordinals can be thought of as a method for labeling and tracking digital collectibles, while OP\_CAT is more focused on improving the transaction capabilities themselves.

The final word
--------------

Bitcoin has evolved beyond its original purpose as a decentralized virtual currency to now support actions such as the creation and transfer of NFTs. With the Bitcoin Improvement Proposal allowing developers to propose further changes, the network's evolution could continue, particularly with OP\_CAT potentially making a return to developers' toolkits.

With OP\_CAT introducing more advanced scripting abilities and, as a result, the ability to create more complex smart contracts, we could soon see fresh possibilities for the Bitcoin network.