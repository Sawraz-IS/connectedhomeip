# Joint Fabric Feature Documentation

- [Overview](#overview)
- [Architecture](#architecture)
  - [Key Roles](#key-roles)
  - [Component Diagram](#component-diagram)
- [Core Components](#core-components)
  - [Joint Fabric Datastore](#joint-fabric-datastore)
  - [Joint Fabric Administrator](#joint-fabric-administrator)
  - [JCM Trust Verification](#jcm-trust-verification)
- [Clusters](#clusters)
  - [Joint Fabric Datastore Cluster (0x0752)](#joint-fabric-datastore-cluster-0x0752)
  - [Joint Fabric Administrator Cluster (0x0753)](#joint-fabric-administrator-cluster-0x0753)
- [Joint Commissioning Method (JCM)](#joint-commissioning-method-jcm)
  - [Commissioning Flow Overview](#commissioning-flow-overview)
  - [Stage 1: Controller-to-Admin (Standard PASE)](#stage-1-controller-to-admin-standard-pase)
  - [Stage 2: Joint Commissioning (JCM)](#stage-2-joint-commissioning-jcm)
  - [Trust Verification Process](#trust-verification-process)
- [CASE Authenticated Tags (CATs)](#case-authenticated-tags-cats)
- [Datastore Synchronization](#datastore-synchronization)
  - [Push Operations (Admin to Node)](#push-operations-admin-to-node)
  - [Pull Operations (Admin from Node)](#pull-operations-admin-from-node)
  - [Node Refresh State Machine](#node-refresh-state-machine)
- [Configuration and Build](#configuration-and-build)
  - [Build-Time Configuration](#build-time-configuration)
  - [Runtime Configuration](#runtime-configuration)
- [Example Applications](#example-applications)
  - [jf-control-app](#jf-control-app)
  - [jf-admin-app](#jf-admin-app)
- [Datastore Limits](#datastore-limits)
- [Source Code Reference](#source-code-reference)
- [Related Documentation](#related-documentation)

---

## Overview

Joint Fabric is a Matter specification feature that enables **multi-ecosystem
device administration** on a shared fabric. It allows multiple independent
vendor ecosystems (e.g., different smart home platforms) to co-administer the
same set of devices without requiring each ecosystem to maintain a separate
fabric.

### Problems Solved

- **Multi-Admin Support**: Devices can be administered by multiple independent
  vendors/ecosystems simultaneously on a single fabric.
- **Ecosystem Interoperability**: Enables seamless co-administration where
  ecosystems trust each other through Vendor ID verification.
- **Centralized State Management**: Provides a unified datastore for managing
  nodes, groups, access control, key sets, and bindings across all
  administrators on the fabric.
- **Secure Onboarding**: The Joint Commissioning Method (JCM) implements a
  secure commissioning procedure for adding new administrators to an existing
  fabric.

---

## Architecture

### Key Roles

| Role                       | Description                                                                                   |
| -------------------------- | --------------------------------------------------------------------------------------------- |
| **Anchor Administrator**   | The primary/first ecosystem admin commissioned onto the fabric. Holds the Anchor CAT.         |
| **Administrator**          | Any ecosystem admin on the fabric (including the anchor). Holds an Administrator CAT.         |
| **Controller (jf-control-app)** | Acts as the commissioner and PKI provider. Orchestrates initial commissioning.           |
| **Admin (jf-admin-app)**   | Holds instances of the JF Datastore and JF Administrator clusters. Manages device lifecycle.  |
| **End Device**             | Standard Matter device (e.g., lighting-app) commissioned onto the joint fabric.               |

### Component Diagram

```
┌─────────────────────┐       ┌─────────────────────┐
│   jf-control-app    │       │   jf-control-app    │
│   (Ecosystem A)     │       │   (Ecosystem B)     │
│   Commissioner/PKI  │       │   Commissioner/PKI  │
└────────┬────────────┘       └────────┬────────────┘
         │ RPC                          │ RPC
         ▼                              ▼
┌─────────────────────┐       ┌─────────────────────┐
│   jf-admin-app      │◄─────►│   jf-admin-app      │
│   (Ecosystem A)     │  JCM  │   (Ecosystem B)     │
│   Anchor Admin      │       │   Peer Admin        │
│                     │       │                     │
│ ┌─────────────────┐ │       │ ┌─────────────────┐ │
│ │ JF Datastore    │ │       │ │ JF Datastore    │ │
│ │ Cluster (0x0752)│ │       │ │ Cluster (0x0752)│ │
│ ├─────────────────┤ │       │ ├─────────────────┤ │
│ │ JF Admin        │ │       │ │ JF Admin        │ │
│ │ Cluster (0x0753)│ │       │ │ Cluster (0x0753)│ │
│ └─────────────────┘ │       │ └─────────────────┘ │
└────────┬────────────┘       └─────────────────────┘
         │
         ▼
┌─────────────────────┐
│   End Devices       │
│   (lighting-app,    │
│    sensors, etc.)   │
└─────────────────────┘
```

---

## Core Components

### Joint Fabric Datastore

**Header**: `src/app/server/JointFabricDatastore.h`
**Implementation**: `src/app/server/JointFabricDatastore.cpp`

The `JointFabricDatastore` is a singleton class that serves as the central
in-memory database for all Joint Fabric state. It tracks every entity on the
fabric: nodes, administrators, groups, key sets, endpoints, bindings, and
access control lists.

#### Core Attributes

| Attribute         | Type       | Description                                       |
| ----------------- | ---------- | ------------------------------------------------- |
| `AnchorRootCA`    | `ByteSpan` | Root CA certificate of the anchor ecosystem       |
| `AnchorNodeID`    | `NodeId`   | Node ID of the anchor administrator               |
| `AnchorVendorID`  | `VendorId` | Vendor ID of the anchor ecosystem                 |
| `FriendlyName`    | `CharSpan` | Human-readable name for the datastore             |
| `Status`          | Struct     | Current state (Pending, Committed, etc.) with timestamp and failure code |

#### Managed Entity Lists

| Entity List            | Description                                          |
| ---------------------- | ---------------------------------------------------- |
| `NodeInformationEntries` | All commissioned nodes with state and friendly name |
| `AdminEntries`         | All administrators with node ID, vendor ID, and ICAC |
| `GroupInformationEntries` | Group definitions with CATs and permissions        |
| `GroupKeySetList`      | Encryption key sets (up to 3 epoch keys per set)     |
| `NodeKeySetEntries`    | Mapping of nodes to their group key sets             |
| `EndpointEntries`      | Endpoints registered on each node                    |
| `EndpointGroupIDEntries` | Group memberships per endpoint                     |
| `EndpointBindingEntries` | Binding targets per endpoint                       |
| `ACLEntries`           | Access control lists per node                        |

#### Key API Methods

```cpp
// Node lifecycle
CHIP_ERROR AddPendingNode(NodeId nodeId, const CharSpan & friendlyName);
CHIP_ERROR UpdateNode(NodeId nodeId, const CharSpan & friendlyName);
CHIP_ERROR RemoveNode(NodeId nodeId);
CHIP_ERROR RefreshNode(NodeId nodeId);  // Async pull of latest state from device

// Administrator management
CHIP_ERROR AddAdmin(AdminInformationEntryStruct & adminId);
CHIP_ERROR UpdateAdmin(NodeId nodeId, CharSpan friendlyName, ByteSpan icac);
CHIP_ERROR RemoveAdmin(NodeId nodeId);

// Group management
CHIP_ERROR AddGroup(const AddGroup::DecodableType & commandData);
CHIP_ERROR UpdateGroup(const UpdateGroup::DecodableType & commandData);
CHIP_ERROR RemoveGroup(const RemoveGroup::DecodableType & commandData);

// Group key set management
CHIP_ERROR AddGroupKeySetEntry(const DatastoreGroupKeySetStruct::Type & groupKeySet);
CHIP_ERROR UpdateGroupKeySetEntry(DatastoreGroupKeySetStruct::Type & groupKeySet);
CHIP_ERROR RemoveGroupKeySetEntry(uint16_t groupKeySetId);

// Endpoint and binding management
CHIP_ERROR UpdateEndpointForNode(NodeId nodeId, EndpointId endpointId, CharSpan friendlyName);
CHIP_ERROR AddGroupIDToEndpointForNode(NodeId nodeId, EndpointId endpointId, GroupId groupId);
CHIP_ERROR RemoveGroupIDFromEndpointForNode(NodeId nodeId, EndpointId endpointId, GroupId groupId);
CHIP_ERROR AddBindingToEndpointForNode(NodeId nodeId, EndpointId endpointId, const BindingTarget & binding);
CHIP_ERROR RemoveBindingFromEndpointForNode(uint16_t listId, NodeId nodeId, EndpointId endpointId);

// ACL management
CHIP_ERROR AddACLToNode(NodeId nodeId, const AccessControlEntry & aclEntry);
CHIP_ERROR RemoveACLFromNode(uint16_t listId, NodeId nodeId);
```

#### Listener Interface

The datastore supports a `Listener` interface to notify observers of state
changes:

```cpp
class Listener {
public:
    virtual void MarkNodeListChanged() = 0;
};

void AddListener(Listener & listener);
void RemoveListener(Listener & listener);
```

### Joint Fabric Administrator

**Header**: `src/app/server/JointFabricAdministrator.h`
**Implementation**: `src/app/server/JointFabricAdministrator.cpp`

The `JointFabricAdministrator` is a singleton that manages administrator-level
state for the JF Administrator cluster. It tracks:

- **Peer JF Admin Cluster Endpoint ID**: The endpoint on the peer admin where
  the JFA cluster is hosted.
- **VID Verification Fabric Index**: Records which fabric has completed Vendor
  ID verification (used during JCM).

#### Delegate Interface

```cpp
class Delegate {
public:
    virtual CHIP_ERROR GetIcacCsr(MutableByteSpan & icacCsr) { return CHIP_NO_ERROR; }
};
```

The delegate allows the application to provide ICAC CSR (Certificate Signing
Request) generation logic, which is used during the JCM process when a peer
administrator requests an intermediate certificate.

### JCM Trust Verification

**Header**: `src/credentials/jcm/TrustVerification.h`
**Related**: `src/credentials/jcm/VendorIdVerificationClient.h`

The trust verification module implements the security validation that occurs
during the Joint Commissioning Method. It maintains a `TrustVerificationInfo`
struct that stores:

- Admin endpoint ID and fabric index
- Admin vendor ID and fabric ID
- Root public key, RCAC, ICAC, and NOC of the peer administrator

#### Error Codes

| Error                              | Code | Description                                    |
| ---------------------------------- | ---- | ---------------------------------------------- |
| `kSuccess`                         | 0    | Trust verification succeeded                   |
| `kAsync`                           | 1    | Verification is in progress                    |
| `kInvalidAdministratorEndpointId`  | 100  | Invalid admin endpoint                         |
| `kInvalidAdministratorFabricIndex` | 101  | Invalid admin fabric index                     |
| `kInvalidAdministratorCAT`         | 102  | Invalid administrator CAT in certificate       |
| `kTrustVerificationDelegateNotSet` | 103  | No delegate configured                         |
| `kUserDeniedConsent`               | 104  | User declined the commissioning consent prompt |
| `kVendorIdVerificationFailed`      | 105  | Vendor ID did not match expectations           |
| `kReadAdminAttributeFailed`        | 106  | Could not read admin fabric attributes         |
| `kAdministratorIdMismatched`       | 107  | Administrator IDs do not match                 |

---

## Clusters

### Joint Fabric Datastore Cluster (0x0752)

**Implementation**: `src/app/clusters/joint-fabric-datastore-server/`
**Cluster XML**: `src/app/zap-templates/zcl/data-model/chip/joint-fabric-datastore.xml`

This cluster exposes the full datastore API over the Matter data model. All
commands require **Administer** privilege.

#### Commands

| Command                            | Direction      | Description                                  |
| ---------------------------------- | -------------- | -------------------------------------------- |
| `AddKeySet`                        | Client→Server  | Add a group key set                          |
| `UpdateKeySet`                     | Client→Server  | Update an existing group key set             |
| `RemoveKeySet`                     | Client→Server  | Remove a group key set by ID                 |
| `AddGroup`                         | Client→Server  | Add a group definition                       |
| `UpdateGroup`                      | Client→Server  | Update a group definition                    |
| `RemoveGroup`                      | Client→Server  | Remove a group by ID                         |
| `AddAdmin`                         | Client→Server  | Add an administrator entry                   |
| `UpdateAdmin`                      | Client→Server  | Update an administrator entry                |
| `RemoveAdmin`                      | Client→Server  | Remove an administrator entry                |
| `AddPendingNode`                   | Client→Server  | Add a node in pending state                  |
| `RefreshNode`                      | Client→Server  | Pull latest state from a remote node         |
| `UpdateNode`                       | Client→Server  | Update a node's friendly name                |
| `RemoveNode`                       | Client→Server  | Remove a node from the datastore             |
| `UpdateEndpointForNode`            | Client→Server  | Update an endpoint's metadata                |
| `AddGroupIDToEndpointForNode`      | Client→Server  | Associate a group with an endpoint           |
| `RemoveGroupIDFromEndpointForNode` | Client→Server  | Remove a group from an endpoint              |
| `AddBindingToEndpointForNode`      | Client→Server  | Add a binding target to an endpoint          |
| `RemoveBindingFromEndpointForNode` | Client→Server  | Remove a binding from an endpoint            |
| `AddACLToNode`                     | Client→Server  | Add an ACL entry to a node                   |
| `RemoveACLFromNode`                | Client→Server  | Remove an ACL entry from a node              |

#### Attributes

All datastore entity lists (nodes, admins, groups, key sets, endpoints,
bindings, ACLs) are exposed as cluster attributes for read access.

### Joint Fabric Administrator Cluster (0x0753)

**Implementation**: `src/app/clusters/joint-fabric-administrator-server/`
**Cluster XML**: `src/app/zap-templates/zcl/data-model/chip/joint-fabric-administrator.xml`

This cluster handles commissioning operations and cross-admin trust
establishment.

#### Commands

| Command                            | Direction      | Description                                         |
| ---------------------------------- | -------------- | --------------------------------------------------- |
| `OpenJointCommissioningWindow`     | Client→Server  | Open a commissioning window for a peer admin        |
| `ICACCSRRequest`                   | Client→Server  | Request a CSR for issuing an ICAC                   |
| `ICACCSRResponse`                  | Server→Client  | Response containing the generated ICAC CSR          |
| `AddICAC`                          | Client→Server  | Submit a signed ICAC to the admin                   |
| `ICACResponse`                     | Server→Client  | Status response for the AddICAC command             |
| `TransferAnchorRequest`            | Client→Server  | Request anchor role transfer                        |
| `TransferAnchorResponse`           | Server→Client  | Status response for anchor transfer                 |
| `TransferAnchorComplete`           | Client→Server  | Finalize anchor role transfer                       |
| `AnnounceJointFabricAdministrator` | Client→Server  | Announce the peer admin's JFA cluster endpoint      |

#### Attributes

| Attribute                    | Type          | Description                                   |
| ---------------------------- | ------------- | --------------------------------------------- |
| `AdministratorFabricIndex`   | `FabricIndex` | The fabric index this administrator operates on |

#### JCMCommissionee

**Header**: `src/app/clusters/joint-fabric-administrator-server/JCMCommissionee.h`

Handles the JCM commissioning flow from the commissionee (target admin) side.
Manages the state machine for verifying the incoming administrator and
establishing trust.

---

## Joint Commissioning Method (JCM)

### Commissioning Flow Overview

JCM is a two-stage process that enables a new ecosystem administrator to join
an existing joint fabric securely.

### Stage 1: Controller-to-Admin (Standard PASE)

1. The **jf-control-app** (controller) performs standard PASE commissioning with
   the **jf-admin-app**.
2. The controller issues a NOC containing both an **Administrator CAT** and an
   **Anchor CAT**.
3. The `CaseAdminSubject` field in `AddNOC` is set to the Administrator CAT.
4. The admin app becomes the **Anchor Administrator** of the fabric.

### Stage 2: Joint Commissioning (JCM)

When a second ecosystem wants to join an existing fabric:

1. **Open Joint Commissioning Window**: The anchor admin's controller calls
   `OpenJointCommissioningWindow` on the JF Administrator cluster, providing
   PAKE passcode verifier, discriminator, iterations, and salt. The admin app
   begins advertising with the `jf` (Joint Fabric) flag.

2. **Peer Admin Initiates JCM**: The second ecosystem's controller uses:
   ```
   pairing code <node-id> <manual-pairing-code> --jcm true
   ```
   This triggers PASE with the target admin, followed by an `ICACCSRRequest` to
   obtain a CSR for issuing an intermediate certificate.

3. **Trust Verification**: The JCMCommissionee module on the target admin
   performs Vendor ID verification, cross-checks administrator identities, and
   requests user consent.

4. **Finalize with AddICAC**: The peer admin submits its signed ICAC via the
   `AddICAC` command. A NOC with the **Anchor CAT** (but not the Administrator
   CAT) is issued, granting the peer admin co-administration rights on the
   shared fabric.

### Trust Verification Process

During JCM, the commissionee (target admin) performs the following checks:

1. **Read Administrator Fabric Index** from the peer admin.
2. **Verify Vendor ID** through the fabric table against the expected vendor.
3. **Validate Administrator CATs** in the peer's certificate chain.
4. **Cross-check Administrator IDs** to ensure consistency.
5. **Request User Consent** before granting access.
6. **Store Peer Endpoint ID** for future JFA cluster communication.

---

## CASE Authenticated Tags (CATs)

Joint Fabric uses two types of CATs embedded in NOC certificates to control
privilege levels:

| CAT Type            | Pattern      | Example      | Description                           |
| ------------------- | ------------ | ------------ | ------------------------------------- |
| **Administrator CAT** | `0xFFFF00xx` | `0xFFFF0001` | Identifies an administrator on the fabric. Only the anchor admin receives this in addition to the Anchor CAT. |
| **Anchor CAT**      | `0xFFFE00xx` | `0xFFFE0001` | Identifies the anchor administrator. All admins on the fabric receive this CAT. |

### CAT Usage in NOC Issuance

- **Anchor commissioning** (`--anchor true`): NOC contains both Administrator
  CAT and Anchor CAT. `CaseAdminSubject` is set to the Administrator CAT.
- **JCM commissioning** (`--jcm true`): NOC contains only the Anchor CAT.
  `CaseAdminSubject` is set to the Anchor CAT.
- **Device commissioning**: Standard NOC without Joint Fabric CATs. ACL subjects
  reference the Administrator CAT.

### CAT Validation

When `AddNOC` is called, the Operational Credentials cluster validates:
- The NOC contains the expected CATs.
- If `CaseAdminSubject` is provided, it must be a valid Anchor CAT
  (`0xFFFE00xx` format).

---

## Datastore Synchronization

The datastore uses a **Delegate** pattern for bidirectional synchronization
between the admin and the devices on the fabric.

### Push Operations (Admin to Node)

These methods push configuration from the datastore to a remote node:

```cpp
// Sync group membership to node
CHIP_ERROR SyncNode(NodeId nodeId, const DatastoreEndpointGroupIDEntryStruct & entry, std::function<void()> onSuccess);

// Sync key set associations
CHIP_ERROR SyncNode(NodeId nodeId, const DatastoreNodeKeySetEntryStruct & entry, std::function<void()> onSuccess);

// Sync endpoint bindings
CHIP_ERROR SyncNode(NodeId nodeId, const DatastoreEndpointBindingEntryStruct & entry, std::function<void()> onSuccess);

// Sync access control lists
CHIP_ERROR SyncNode(NodeId nodeId, const std::vector<DatastoreACLEntryStruct> & entries, std::function<void()> onSuccess);

// Sync group encryption keys
CHIP_ERROR SyncNode(NodeId nodeId, const DatastoreGroupKeySetStruct & groupKeySet, std::function<void()> onSuccess);
```

### Pull Operations (Admin from Node)

These methods fetch the current state from a remote node:

```cpp
CHIP_ERROR FetchEndpointList(NodeId nodeId, Callback);
CHIP_ERROR FetchEndpointGroupList(NodeId nodeId, EndpointId endpointId, Callback);
CHIP_ERROR FetchEndpointBindingList(NodeId nodeId, EndpointId endpointId, Callback);
CHIP_ERROR FetchGroupKeySetList(NodeId nodeId, Callback);
CHIP_ERROR FetchACLList(NodeId nodeId, Callback);
```

### Node Refresh State Machine

When `RefreshNode()` is called, the datastore walks through a state machine to
pull all data from the remote node:

```
kIdle → kRefreshingEndpoints → kRefreshingGroups → kRefreshingBindings
      → kFetchingGroupKeySets → kRefreshingGroupKeySets → kRefreshingACLs → kIdle
```

The `ContinueRefresh()` method advances through these states. Each state
triggers the appropriate `Fetch*` delegate call and transitions to the next
state upon completion.

The synchronization implementation for the example applications can be found in
`examples/jf-admin-app/linux/JFADatastoreSync.cpp`.

---

## Configuration and Build

### Build-Time Configuration

Enable Joint Fabric support with the GN build argument:

```bash
gn gen out/test --args="chip_device_config_enable_joint_fabric=true"
```

All Joint Fabric code is guarded by:

```cpp
#if CHIP_DEVICE_CONFIG_ENABLE_JOINT_FABRIC
// Joint Fabric code
#endif
```

### Runtime Configuration

**jf-admin-app command-line options:**

| Flag                    | Description                                |
| ----------------------- | ------------------------------------------ |
| `--capabilities 0x4`   | Set device capabilities for JF             |
| `--passcode`            | PASE passcode for commissioning            |
| `--discriminator`       | BLE/DNS-SD discriminator                   |
| `--secured-device-port` | Port for secure communication              |
| `--rpc-server-port`     | Port for RPC communication with controller |
| `--KVS`                 | Path to key-value store file               |

**jf-control-app command-line options:**

| Flag                        | Description                              |
| --------------------------- | ---------------------------------------- |
| `--rpc-server-port`         | Port for RPC communication with admin    |
| `--storage-directory`       | Directory for persistent storage         |
| `--commissioner-vendor-id`  | Vendor ID for this ecosystem             |

**Commissioning command flags:**

| Flag              | Description                                          |
| ----------------- | ---------------------------------------------------- |
| `--anchor true`   | Commission the target as the anchor administrator    |
| `--jcm true`      | Use Joint Commissioning Method for peer admin join   |

---

## Example Applications

### jf-control-app

**Location**: `examples/jf-control-app/`

Acts as the commissioner and PKI provider for a Joint Fabric ecosystem. It:
- Performs initial PASE commissioning of the jf-admin-app.
- Issues NOCs with the appropriate CATs (Administrator + Anchor for anchor,
  Anchor-only for JCM).
- Commissions end devices through standard PASE, then hands off ownership to
  the jf-admin-app via RPC.

### jf-admin-app

**Location**: `examples/jf-admin-app/`

Acts as the fabric administrator. It:
- Hosts the JF Datastore Cluster (0x0752) and JF Administrator Cluster (0x0753).
- Maintains the central datastore of all nodes, groups, ACLs, and key sets.
- Handles `OpenJointCommissioningWindow` to allow peer admins to join.
- Performs JCM trust verification when acting as the commissionee.
- Synchronizes configuration to end devices via the datastore delegate.

---

## Datastore Limits

| Resource             | Maximum   |
| -------------------- | --------- |
| Nodes                | 256       |
| Admin Nodes          | 32        |
| Groups               | 16        |
| Group Key Sets       | 256       |
| ACL Entries          | 64        |
| Friendly Name Length | 32 chars  |

---

## Source Code Reference

| Component                  | Path                                                                  |
| -------------------------- | --------------------------------------------------------------------- |
| Datastore Core             | `src/app/server/JointFabricDatastore.{h,cpp}`                         |
| Administrator Core         | `src/app/server/JointFabricAdministrator.{h,cpp}`                     |
| Datastore Cluster Server   | `src/app/clusters/joint-fabric-datastore-server/`                     |
| Administrator Cluster Server | `src/app/clusters/joint-fabric-administrator-server/`               |
| JCM Trust Verification     | `src/credentials/jcm/TrustVerification.{h,cpp}`                      |
| VID Verification Client    | `src/credentials/jcm/VendorIdVerificationClient.{h,cpp}`             |
| Cluster XML Definitions    | `src/app/zap-templates/zcl/data-model/chip/joint-fabric-*.xml`        |
| Example Admin App          | `examples/jf-admin-app/`                                              |
| Example Controller App     | `examples/jf-control-app/`                                            |
| Datastore Sync (Example)   | `examples/jf-admin-app/linux/JFADatastoreSync.cpp`                    |
| Unit Tests                 | `src/app/server/tests/TestJointFabricDatastore.cpp`                   |
| Operational Guide          | `docs/guides/joint_fabric_guide.md`                                   |

---

## Related Documentation

- [Joint Fabric Guide](joint_fabric_guide.md) - Step-by-step operational guide
  for running the Joint Fabric demo on Linux.
- [jf-control-app README](../../examples/jf-control-app/README.md) - Build
  instructions for the Joint Fabric Controller application.
- [jf-admin-app README](../../examples/jf-admin-app/linux/README.md) - Build
  instructions for the Joint Fabric Admin application.
- Matter Specification - Joint Commissioning Method (JCM) chapter.
