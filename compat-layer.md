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

The compatibility layer allows developers to reuse existing Fabric chaincodes on Fabric-X while retaining familiar deployment and invocation workflows.

The architecture consists of the following components:

| Component | Responsibility |
|---|---|
| **Orchestrator** | Coordinates execution across organizations, collects endorsements, and submits transactions |
| **Compatibility Helper** | Handles Fabric shim communication and translates chaincode state requests into Fabric-X-compatible operations |
| **Chaincode Service** | Hosts and executes existing Fabric chaincodes as independent CCaaS services |
| **Lifecycle Manager** | Manages chaincode installation, organizational approvals, and committed definitions |
| **Discovery Resolver** | Locates and selects Orchestrator instances through configurable discovery mechanisms |
| **Transaction Store** | Tracks transaction progress and identifies duplicate requests |

## Chaincode execution

Consider an existing Fabric application that uses chaincode to manage assets. With the compatibility layer, developers can deploy the same chaincode as an external service and continue invoking its existing functions.

For example, an asset-transfer operation can continue using familiar chaincode APIs such as `GetState`, `PutState`, and `DelState` to retrieve an asset, change its ownership, and update its state. Developers do not need to rewrite this business logic using Fabric-X-specific transaction APIs.

The compatibility layer handles the underlying interaction with Fabric-X, allowing supported chaincode operations to follow its transaction processing model.

## Multi-organization endorsement

Fabric applications may require multiple organizations to endorse a transaction. The compatibility layer preserves this approach without requiring developers to implement cross-organization execution themselves.

For example, an asset transfer requiring endorsements from Org0 and Org1 can be executed through their respective Orchestrators. The transaction proceeds only when both organizations produce consistent execution results and the required endorsements are obtained. If the results differ, the transaction is rejected.

From the application's perspective, the existing organizational endorsement requirements remain part of the transaction workflow.

## Chaincode lifecycle management

Developers can use familiar lifecycle operations such as `package`, `install`, `approveformyorg`, `checkcommitreadiness`, `commit`, and `querycommitted` to manage their chaincodes.

Once a chaincode definition is committed, newly started Orchestrators within the same organization can recognize the deployment without repeating the commitment process. This simplifies chaincode management across multiple instances.

## Orchestrator discovery and availability

Organizations are not restricted to a single Orchestrator instance. Multiple instances can be deployed to distribute workloads and improve service availability.

The compatibility layer allows these instances to be discovered and selected without modifying existing chaincodes or their endorsement requirements. This provides a foundation for load balancing and flexible deployment configurations.

Repeated transaction requests can also be identified, helping reduce unnecessary execution and submission.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The compatibility layer operates independently of Fabric-X core components and communicates with existing services through their exposed APIs.

## Architectural overview

The architecture separates chaincode execution from Fabric-X transaction processing. Each Orchestrator contains an embedded compatibility helper that implements the Fabric chaincode shim communication protocol and communicates with external Chaincode-as-a-Service (CCaaS) runtimes.

The design separates three main responsibilities:

- **Execution:** The compatibility helper provides the chaincode execution context and translates state operations into transaction simulation results.
- **Coordination:** Orchestrators manage communication between participating organizations and assemble endorsed transactions.
- **State management:** Fabric-X maintains committed ledger state and chaincode definitions, while Orchestrators maintain local runtime and transaction information.

This separation allows the compatibility layer to reuse existing Fabric chaincode execution mechanisms without introducing the traditional Fabric peer architecture into Fabric-X core components.

## Execution and endorsement

When an Orchestrator receives a transaction proposal, it resolves the corresponding chaincode runtime and invokes it through the embedded compatibility helper.

The helper processes Fabric shim messages and translates state requests into operations supported by Fabric-X. Direct key-based reads are retrieved from committed state through the Committer Query Service, while writes and deletions are recorded locally to construct the transaction's read-write set.

For multi-organization endorsement, the initiating Orchestrator resolves the required organizations and requests independent execution through their respective Orchestrators. The resulting execution responses and read-write sets are compared for consistency before endorsements are combined. Any disagreement prevents the transaction from proceeding to submission.

The Fabric-X SDK provides reusable functionality for transaction construction, identity handling, and interaction with Fabric-X services. The resulting transaction is submitted to the Ordering Service, while its commit status is observed through the Notification Service.

## Lifecycle management

The lifecycle manager separates local chaincode installation information from committed chaincode definitions.

Packaging and installation prepare the chaincode runtime, while organizational approvals and commit-readiness checks determine whether a definition can be committed.

Installation records remain local to individual Orchestrators, while committed definitions are persisted on the Fabric-X ledger. These definitions are indexed to support retrieval and reconstruction during startup.

When an Orchestrator starts, it synchronizes its committed lifecycle state from the ledger, independently of the instance that originally committed the chaincode. This enables multiple Orchestrators within the same organization to maintain a consistent view of deployed chaincode definitions.

## Discovery and transaction coordination

The discovery interface supports both statically configured endpoints and dynamic resolution of Orchestrator instances. Each organization can be associated with multiple endpoints, and resolved connections can be cached for subsequent requests.

This allows an Orchestrator to select among available instances when coordinating execution with other organizations. Endpoint selection, load balancing, and failover strategies can evolve independently of the chaincode execution logic.

To reduce redundant processing, the Orchestrator maintains a transaction store with the following states:

- `IN_PROGRESS`: Transaction execution has started.
- `EXECUTED`: Execution results have been collected.
- `SUBMITTED`: Transaction has been submitted for ordering.
- `COMPLETED`: A final transaction outcome has been observed.

These states help identify repeated requests and track execution progress. Additional recovery and synchronization mechanisms may be required when multiple Orchestrators process transactions concurrently or restart during execution.

## Compatibility limitations

The initial implementation supports direct key-based state access, allowing chaincodes to read, write, and delete individual ledger entries using their exact keys.

Operations requiring ordered key traversal, including range queries and pagination, depend on additional capabilities in the Fabric-X Query Service. Private data collections and other advanced query operations remain outside the initial scope.

For further architectural details and implementation considerations, see the [Compatibility Layer Design Document](https://docs.google.com/document/d/1eseyFreUHmNmviQoq8qYGqatzMXgYZb79CFnlk6kBqg/edit?tab=t.0) (working draft).

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
