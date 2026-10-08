---
layout: default
title: Fabric Chaincode Compatibility
nav_order: 3
---

- Feature Name: Fabric Chaincode Compatibility
- Start Date: 2026-10-08
- RFC PR:
- Fabric-X Component: fabric-x-chaincode-compatibility
- Fabric-X Issue:

# Summary
[summary]: #summary

This proposal introduces a compatibility layer that enables existing Hyperledger Fabric chaincodes to run on Fabric-X without modifying their business logic. It provides an execution environment that translates Fabric chaincode operations into Fabric-X transactions, while supporting multi organization endorsement, chaincode lifecycle management, and pluggable orchestrator discovery. The goal is to simplify migration from Fabric to Fabric-X while preserving the existing chaincode programming model.

# Motivation
[motivation]: #motivation

Fabric-X offers better performance and scalability compared to traditional Hyperledger Fabric. However, businesses already using Fabric have invested heavily in their existing chaincodes and infrastructure, making migration to Fabric-X costly and difficult.

To encourage Fabric-X adoption, organizations need a simpler way to migrate without rebuilding their existing applications. The compatibility layer addresses this by allowing existing Fabric chaincodes to work with Fabric-X with minimal changes, reducing migration effort and encouraging wider adoption.

The compatibility layer addresses this gap by reusing the existing Fabric chaincode shim protocol and connecting chaincode execution to Fabric-X services.

The main objectives are:

- Run existing Go chaincodes using Chaincode-as-a-Service (CCaaS).
- Support multi organization execution and endorsement.
- Provide Fabric-compatible chaincode lifecycle operations.
- Allow extensible discovery and integration with Fabric-X components.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The compatibility layer introduces an **Orchestrator** that allows existing Hyperledger Fabric chaincodes to operate on Fabric-X using the familiar Chaincode-as-a-Service (CCaaS) model. It bridges the traditional Fabric execution model with Fabric-X's transaction processing architecture.

The architecture consists of the following components:

| Component | Responsibility |
|---|---|
| Orchestrator | Coordinates execution, endorsement, and transaction submission |
| Compatibility Helper | Handles Fabric shim communication and state operations |
| Chaincode Service | Runs existing Fabric chaincodes through CCaaS |
| Lifecycle Manager | Manages chaincode deployment and definitions |
| Discovery Resolver | Resolves available orchestrators across organizations |
| Transaction Store | Tracks transaction progress and duplicate requests |

## Chaincode execution

Consider an existing Fabric application that uses chaincode to manage assets. Normally, this chaincode executes through a Fabric peer, which provides access to ledger state.

With the compatibility layer, the same chaincode runs as an external service connected to an Orchestrator. When a client invokes a transaction, the embedded compatibility helper communicates with the chaincode using the Fabric shim protocol.

Operations such as `GetState`, `PutState`, and `DelState` are translated into reads and proposed state changes using Fabric-X services. The resulting read-write set is assembled into a Fabric-X transaction, which follows the existing ordering and validation process.

This allows developers to reuse their existing chaincode business logic without adapting it to Fabric-X's internal transaction model.

## Multi-organization endorsement

Fabric applications may require endorsements from multiple organizations before a transaction can be committed.

For example, consider a transaction requiring endorsements from Org0 and Org1. The initiating Orchestrator coordinates with an Orchestrator from the other organization, allowing both to independently execute the same proposal.

Their execution results are compared for consistency, and the required endorsements are collected before the transaction proceeds to ordering. If the results differ, the transaction is rejected.

## Chaincode lifecycle management

The compatibility layer provides familiar lifecycle operations such as `install`, `approveformyorg`, `checkcommitreadiness`, and `commit`.

Chaincode installation remains local to an Orchestrator, while committed definitions are maintained on the Fabric-X ledger. This distinction allows newly started Orchestrators to reconstruct deployed chaincode definitions without repeating the commitment process.

## Orchestrator discovery and availability

An organization can operate multiple Orchestrators rather than relying on a single instance.

The pluggable discovery mechanism supports both statically configured endpoints and dynamic resolution, allowing requests to be directed to an available Orchestrator within the required organization.

Transaction tracking also helps identify repeated requests and avoid redundant processing during execution and submission.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The compatibility layer operates independently of Fabric-X core components and communicates with existing services through their exposed APIs.

## Architectural overview

The architecture is centered around the Orchestrator, which connects Fabric chaincode execution with Fabric-X transaction processing. Each Orchestrator contains an embedded compatibility helper responsible for communicating with external chaincode services through the Fabric shim protocol.

The design separates three main responsibilities:

- **Execution:** Chaincodes run as external CCaaS services, with state operations handled by the embedded helper.
- **Coordination:** Orchestrators manage multi-organization execution, endorsement collection, and transaction submission.
- **State and lifecycle:** Fabric-X maintains committed ledger state and chaincode definitions, while Orchestrators maintain their local runtime information.

Organizations can operate multiple Orchestrators, with a pluggable discovery mechanism used to resolve available instances. This allows chaincode execution to scale independently of Fabric-X core components.

## Execution and endorsement

An incoming proposal is processed by the initiating Orchestrator, which resolves the corresponding chaincode runtime and invokes it through the embedded helper.

State reads are performed through the Fabric-X Query Service, while writes and deletions are captured locally to construct the transaction's read-write set.

For multi-organization endorsement, the initiating Orchestrator coordinates independent executions across the required organizations. Their results are compared for consistency, and the collected endorsements are used to construct a Fabric-X transaction.

The transaction is submitted through the Ordering Service, while its finality is observed through the Notification Service.

## Lifecycle management

The lifecycle manager provides operations compatible with the Fabric chaincode lifecycle, including packaging, installation, organizational approval, readiness checks, and commitment.

The design separates local installation metadata from committed chaincode definitions. Installation records remain specific to each Orchestrator, while committed definitions are persisted on the Fabric-X ledger.

During startup, Orchestrators reconstruct committed lifecycle information from ledger state, enabling newly started instances to recognize existing chaincode deployments without repeating the commitment process.

For further architectural details and implementation considerations, see the [Compatibility Layer Design Document](https://docs.google.com/document/d/1eseyFreUHmNmviQoq8qYGqatzMXgYZb79CFnlk6kBqg/edit?tab=t.0) (working draft).
## Lifecycle management

The lifecycle manager provides operations compatible with the Fabric chaincode lifecycle, including packaging, installation, organizational approval, readiness checks, and commitment.

Local installation information is maintained by individual orchestrators, while committed chaincode definitions are stored on the Fabric-X ledger.

During startup, an orchestrator reconstructs committed lifecycle information from ledger state, allowing multiple instances within the same organization to share a consistent view of deployed chaincodes.

## Discovery and transaction coordination

A pluggable discovery interface allows orchestrators to resolve other instances using static configuration or dynamic resolution.

Each organization may expose multiple orchestrator endpoints, enabling instance selection and future load-balancing strategies without changing the endorsement flow.

A transaction store tracks requests through `IN_PROGRESS`, `EXECUTED`, `SUBMITTED`, and `COMPLETED` states to reduce redundant execution and submission.

## Compatibility limitations

The initial implementation supports direct key-based state access, allowing chaincodes to read, write, and delete individual entries using their exact keys,
Advanced operations such as range queries and pagination depend on additional Fabric-X Query Service capabilities. Private data and some advanced query operations remain outside the initial scope.

# Drawbacks
[drawbacks]: #drawbacks

The compatibility layer introduces additional services and communication overhead, which may increase transaction latency and limit performance compared with applications interacting directly with Fabric-X.

The initial implementation supports only the Chaincode-as-a-Service (CCaaS) model, requiring chaincodes to run as external services.

Maintaining compatibility with Fabric chaincode protocols introduces ongoing maintenance requirements.

Some Fabric features, particularly private data collections and advanced queries, cannot be supported without additional functionality or architectural changes.

# Rationale and alternatives
[alternatives]: #alternatives

An alternative is to rewrite existing chaincodes as native Fabric-X applications. While this offers flexibility, it imposes significant migration costs.

Another approach would be to integrate chaincode execution directly into Fabric-X core components. However, this would introduce a dependency on the traditional Fabric execution model.

The proposed approach keeps chaincode execution separate from Fabric-X, allowing existing applications to migrate while preserving the modular architecture of Fabric-X.

# Prior art
[prior-art]: #prior-art

- [Fabric Chaincode-as-a-Service](https://hyperledger-fabric.readthedocs.io/en/latest/cc_service.html) demonstrates how chaincode execution can be separated from the Fabric peer process.
- [Fabric Smart Client](https://github.com/hyperledger-labs/fabric-smart-client) provides an alternative model for coordinating distributed transaction execution.
- The Fabric-X SDK proposal introduces reusable building blocks for custom endorsers and identifies chaincode compatibility as an intended application.

# Testing
[testing]: #testing

Beyond standard unit tests for individual components such as the Orchestrator, compatibility helper, and lifecycle manager, validating this proposal requires integration and end-to-end testing to ensure correctness and compatibility.

The primary goal is to demonstrate that existing Fabric chaincodes can execute on Fabric-X without modifying their business logic, while preserving consistent execution results, endorsement, and commit semantics. To achieve this, we will adopt two complementary strategies:

1. **Replay Existing Chaincodes:** Select representative chaincodes from the [Fabric Samples](https://github.com/hyperledger/fabric-samples) repository and execute their supported operations on Fabric-X. Verify that state changes, transaction responses, and chaincode events behave as expected, including multi-organization endorsement and MVCC validation.

2. **Lifecycle and Orchestrator Integration:** Test chaincode installation, organizational approvals, and ledger-backed definitions across multiple Orchestrators. Verify that newly started instances can reconstruct lifecycle state, discover other Orchestrators, and handle duplicate requests and transaction finality correctly.

The success criteria are that existing Fabric chaincodes produce consistent execution results across participating organizations and that the resulting transactions are correctly validated and committed by Fabric-X.

These tests will run end-to-end against a Fabric-X network, covering both normal execution and failure scenarios. The goal is to demonstrate that the compatibility layer provides a reliable migration path for existing Fabric applications without requiring changes to their chaincode business logic.

# Dependencies
[dependencies]: #dependencies

The compatibility layer depends on:

- Fabric chaincode shim and protobuf definitions.
- [Fabric-X SDK](https://github.com/hyperledger/fabric-x-sdk) for transaction construction, identity handling, and interaction with Fabric-X services.
- Fabric-X Committer Query Service.
- Fabric-X Ordering and Notification Services.
- gRPC for orchestrator and chaincode communication.

Additional Query Service functionality will be required for complete range-query and iterator support.

# Unresolved questions
[unresolved]: #unresolved-questions

- Should Fabric's package and install lifecycle steps be simplified for CCaaS deployments?
- How should range query support be introduced into Fabric-X?
- What additional coordination is required for reliable transaction recovery across multiple orchestrators?
- How should authentication and authorization be enforced between orchestrators?
- Which additional Fabric chaincode APIs should be prioritized for future compatibility?
