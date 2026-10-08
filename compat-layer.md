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

This proposal introduces a compatibility layer that enables existing Hyperledger Fabric chaincodes to run on Fabric-X without modifying their business logic. It provides an execution environment that translates Fabric chaincode operations into Fabric-X transactions, while supporting multi-organization endorsement, chaincode lifecycle management, and pluggable orchestrator discovery. The goal is to simplify migration from Fabric to Fabric-X while preserving the existing chaincode programming model.

# Motivation
[motivation]: #motivation

Fabric-X does not include the traditional Fabric chaincode execution model. Instead, applications are responsible for constructing and submitting transactions through external components.

While this provides greater flexibility, organizations with existing Fabric chaincodes must rewrite their applications or develop custom execution environments before migrating to Fabric-X.

The compatibility layer addresses this gap by reusing the existing Fabric chaincode shim protocol and connecting chaincode execution to Fabric-X services.

The main objectives are:

- Run existing Go chaincodes using Chaincode-as-a-Service (CCaaS).
- Support multi-organization execution and endorsement.
- Provide Fabric-compatible chaincode lifecycle operations.
- Support multiple orchestrators per organization for load balancing.
- Allow extensible discovery and integration with Fabric-X components.

This provides a migration path for existing Fabric applications without introducing chaincode execution into Fabric-X core services.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

The compatibility layer introduces an **Orchestrator**, which coordinates chaincode execution and transaction submission on Fabric-X.

Existing chaincodes continue to run as external services using the Fabric shim protocol. The Orchestrator communicates with these services through an embedded compatibility helper and translates their state operations into Fabric-X read-write sets.

The architecture consists of the following components:

| Component | Responsibility |
|---|---|
| Orchestrator | Coordinates execution, endorsement, and submission |
| Compatibility Helper | Handles Fabric shim messages and state operations |
| Chaincode Service | Executes existing Fabric chaincode through CCaaS |
| Lifecycle Manager | Manages installation, approvals, and committed definitions |
| Discovery Resolver | Resolves available orchestrators across organizations |
| Transaction Store | Tracks execution progress and duplicate requests |

For example, when a client invokes an asset-transfer chaincode, the Orchestrator forwards the request to the chaincode service. Operations such as `GetState`, `PutState`, and `DelState` are handled by the compatibility helper using the Fabric-X Query Service and a local simulation store.

The resulting read-write set is then assembled into a Fabric-X transaction and submitted to the ordering service.

When multiple organizations are required for endorsement, their orchestrators execute the same proposal independently. The results are compared before endorsements are collected and the transaction is submitted.

Developers can also use familiar lifecycle operations such as `install`, `approveformyorg`, `commit`, and `querycommitted`. Committed definitions are stored on the Fabric-X ledger, allowing newly started orchestrators to reconstruct lifecycle state.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

The compatibility layer operates independently of Fabric-X core components and communicates with existing services through their exposed APIs.

## Execution and endorsement

An incoming proposal is processed by the initiating Orchestrator, which resolves the corresponding chaincode runtime and invokes it through the embedded helper.

The helper implements the Fabric chaincode communication protocol, captures state operations, and constructs the simulation results.

Reads are performed against committed state through the Fabric-X Query Service, while writes and deletions are accumulated for transaction construction.

For multi-organization endorsement, the initiating Orchestrator resolves the required organizations and coordinates independent executions. Matching results are combined with the required endorsements before submission.

The transaction is submitted to the Fabric-X Ordering Service, and completion is observed through the Notification Service.

## Lifecycle management

The lifecycle manager provides operations compatible with the Fabric chaincode lifecycle, including packaging, installation, organizational approval, readiness checks, and commitment.

Local installation information is maintained by individual orchestrators, while committed chaincode definitions are stored on the Fabric-X ledger.

During startup, an orchestrator reconstructs committed lifecycle information from ledger state, allowing multiple instances within the same organization to share a consistent view of deployed chaincodes.

## Discovery and transaction coordination

A pluggable discovery interface allows orchestrators to resolve other instances using static configuration or dynamic resolution.

Each organization may expose multiple orchestrator endpoints, enabling instance selection and future load-balancing strategies without changing the endorsement flow.

A transaction store tracks requests through `IN_PROGRESS`, `EXECUTED`, `SUBMITTED`, and `COMPLETED` states to reduce redundant execution and submission.

## Compatibility limitations

The initial implementation supports the primary Fabric state operations and composite-key functionality.

Advanced operations such as range queries and pagination depend on additional Fabric-X Query Service capabilities. Private data and some advanced query operations remain outside the initial scope.

# Drawbacks
[drawbacks]: #drawbacks

The compatibility layer introduces additional services and communication overhead compared with applications that interact directly with Fabric-X.

Maintaining compatibility with Fabric chaincode protocols also introduces ongoing maintenance requirements.

Some Fabric features, particularly private data collections and advanced queries, cannot be supported without additional functionality or architectural changes.

# Rationale and alternatives
[alternatives]: #alternatives

An alternative is to rewrite existing chaincodes as native Fabric-X applications. While this offers flexibility, it imposes significant migration costs.

Another approach would be to integrate chaincode execution directly into Fabric-X core components. However, this would introduce a dependency on the traditional Fabric execution model.

The proposed approach keeps chaincode execution separate from Fabric-X, allowing existing applications to migrate while preserving the modular architecture of Fabric-X.

# Prior art
[prior-art]: #prior-art

- [Hyperledger Fabric](https://github.com/hyperledger/fabric) provides the chaincode execution protocol and lifecycle model used by this proposal.
- [Fabric Chaincode-as-a-Service](https://hyperledger-fabric.readthedocs.io/en/latest/cc_service.html) demonstrates how chaincode execution can be separated from the Fabric peer process.
- [Fabric Smart Client](https://github.com/hyperledger-labs/fabric-smart-client) provides an alternative model for coordinating distributed transaction execution.
- The Fabric-X SDK proposal introduces reusable building blocks for custom endorsers and identifies chaincode compatibility as an intended application.

# Testing
[testing]: #testing

Testing will include:

- Running existing Fabric sample chaincodes without business-logic modifications.
- Validating single-organization and multi-organization endorsement.
- Verifying consistency checks when organizations produce different execution results.
- Testing lifecycle operations, ledger persistence, and restart recovery.
- Testing multiple orchestrators within the same organization.
- Validating duplicate transaction handling and finality tracking.

Integration tests will execute transactions against a running Fabric-X network.

# Dependencies
[dependencies]: #dependencies

The compatibility layer depends on:

- Fabric chaincode shim and protobuf definitions.
- Fabric-X Committer Query Service.
- Fabric-X Ordering and Notification Services.
- gRPC for orchestrator and chaincode communication.

Additional Query Service functionality will be required for complete range-query and iterator support.

# Unresolved questions
[unresolved]: #unresolved-questions

- Should Fabric's package and install lifecycle steps be simplified for CCaaS deployments?
- How should range-query and iterator support be introduced into Fabric-X?
- What additional coordination is required for reliable transaction recovery across multiple orchestrators?
- How should authentication and authorization be enforced between orchestrators?
- Which additional Fabric chaincode APIs should be prioritized for future compatibility?