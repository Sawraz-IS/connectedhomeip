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

The feature is gated behind a compile-time flag. From
`examples/jf-admin-app/linux/include/CHIPProjectAppConfig.h`:

```cpp
// Device type 0x0130 (304) = Joint Fabric Administrator
#define CHIP_DEVICE_CONFIG_DEVICE_TYPE 304

#define CHIP_DEVICE_CONFIG_ENABLE_BOTH_COMMISSIONER_AND_COMMISSIONEE 1
#define CHIP_DEVICE_CONFIG_DEVICE_NAME "JF Admin"
```

Enable Joint Fabric at build time:

```bash
gn gen out/test --args="chip_device_config_enable_joint_fabric=true"
```

All Joint Fabric code paths are guarded by:

```cpp
#if CHIP_DEVICE_CONFIG_ENABLE_JOINT_FABRIC
// ...
#endif
```

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

#### Key API Methods with Implementation Details

**AddPendingNode** — from `src/app/server/JointFabricDatastore.cpp`:

```cpp
CHIP_ERROR JointFabricDatastore::AddPendingNode(NodeId nodeId, const CharSpan & friendlyName)
{
    VerifyOrReturnError(mNodeInformationEntries.size() < kMaxNodes, CHIP_ERROR_NO_MEMORY);
    // check that nodeId does not already exist
    VerifyOrReturnError(
        std::none_of(mNodeInformationEntries.begin(), mNodeInformationEntries.end(),
                     [nodeId](const GenericDatastoreNodeInformationEntry & entry) { return entry.nodeID == nodeId; }),
        CHIP_IM_GLOBAL_STATUS(ConstraintError));

    mNodeInformationEntries.push_back(GenericDatastoreNodeInformationEntry(
        nodeId, Clusters::JointFabricDatastore::DatastoreStateEnum::kPending, MakeOptional(friendlyName)));

    for (Listener * listener = mListeners; listener != nullptr; listener = listener->mNext)
    {
        listener->MarkNodeListChanged();
    }

    return CHIP_NO_ERROR;
}
```

**AddAdmin** — validates uniqueness, enforces capacity limits, and copies
data into owned storage:

```cpp
CHIP_ERROR JointFabricDatastore::AddAdmin(
    Clusters::JointFabricDatastore::Structs::DatastoreAdministratorInformationEntryStruct::Type & adminId)
{
    VerifyOrReturnError(IsAdminEntryPresent(adminId.nodeID) == false, CHIP_IM_GLOBAL_STATUS(ConstraintError));
    VerifyOrReturnError(mAdminEntries.size() < kMaxAdminNodes, CHIP_ERROR_NO_MEMORY);

    ReturnErrorOnFailure(SetAdminEntryWithOwnedStorage(adminId.nodeID, adminId.friendlyName, adminId.icac, adminId));
    mAdminEntries.push_back(adminId);

    return CHIP_NO_ERROR;
}
```

**AddGroup** — prevents adding groups with reserved Admin/Anchor CAT
identifiers:

```cpp
CHIP_ERROR JointFabricDatastore::AddGroup(
    const Clusters::JointFabricDatastore::Commands::AddGroup::DecodableType & commandData)
{
    size_t index = 0;
    VerifyOrReturnError(IsGroupIDInDatastore(commandData.groupID, index) == CHIP_ERROR_NOT_FOUND,
                        CHIP_IM_GLOBAL_STATUS(ConstraintError));

    if (commandData.groupCAT.ValueOr(0) == kAdminCATIdentifier ||
        commandData.groupCAT.ValueOr(0) == kAnchorCATIdentifier)
    {
        return CHIP_IM_GLOBAL_STATUS(ConstraintError);
    }

    // ... populate groupEntry fields, store with owned storage ...
    mGroupInformationEntries.push_back(groupEntry);
    return CHIP_NO_ERROR;
}
```

**AddACLToNode** — deduplicates existing ACL entries and synchronizes
the new entry to the remote device through the delegate:

```cpp
CHIP_ERROR JointFabricDatastore::AddACLToNode(
    NodeId nodeId,
    const Clusters::JointFabricDatastore::Structs::DatastoreAccessControlEntryStruct::DecodableType & aclEntry)
{
    VerifyOrReturnError(mDelegate != nullptr, CHIP_ERROR_INCORRECT_STATE);

    size_t index = 0;
    ReturnErrorOnFailure(IsNodeIdInNodeInformationEntries(nodeId, index));

    // Check if the ACL entry already exists for the node
    for (auto & entry : mACLEntries)
    {
        if (entry.nodeID == nodeId && ACLMatches(entry.ACLEntry, aclEntry))
            return CHIP_NO_ERROR;
    }
    VerifyOrReturnError(mACLEntries.size() < kMaxACLs, CHIP_ERROR_NO_MEMORY);

    // ... create newACLEntry, populate subjects/targets ...

    return mDelegate->SyncNode(nodeId, entryToEncode, [this]() {
        mACLEntries.back().statusEntry.state =
            Clusters::JointFabricDatastore::DatastoreStateEnum::kCommitted;
    });
}
```

**RefreshNode** — initiates the async state machine that pulls all data
from a remote node:

```cpp
CHIP_ERROR JointFabricDatastore::RefreshNode(NodeId nodeId)
{
    VerifyOrReturnError(mDelegate != nullptr, CHIP_ERROR_INCORRECT_STATE);
    VerifyOrReturnError(mRefreshingNodeId == kUndefinedNodeId, CHIP_ERROR_INCORRECT_STATE);
    VerifyOrReturnError(mRefreshState == kIdle, CHIP_ERROR_INCORRECT_STATE);

    mRefreshingNodeId = nodeId;
    ReturnErrorOnFailure(ContinueRefresh());
    return CHIP_NO_ERROR;
}
```

The full list of public methods on `JointFabricDatastore`:

```cpp
// Node lifecycle
CHIP_ERROR AddPendingNode(NodeId nodeId, const CharSpan & friendlyName);
CHIP_ERROR UpdateNode(NodeId nodeId, const CharSpan & friendlyName);
CHIP_ERROR RemoveNode(NodeId nodeId);
CHIP_ERROR RefreshNode(NodeId nodeId);

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
changes. When the node list changes, all registered listeners are called:

```cpp
class Listener {
public:
    virtual ~Listener() = default;
    virtual void MarkNodeListChanged() = 0;
};

void AddListener(Listener & listener);
void RemoveListener(Listener & listener);
```

The cluster server registers itself as a listener and triggers Matter attribute
change reporting — from `joint-fabric-datastore-server.cpp`:

```cpp
void JointFabricDatastoreAttrAccess::MarkNodeListChanged()
{
    MatterReportingAttributeChangeCallback(kRootEndpointId,
        JointFabricDatastoreCluster::Id,
        JointFabricDatastoreCluster::Attributes::NodeList::Id);
}
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

From `src/app/server/JointFabricAdministrator.h`:

```cpp
class JointFabricAdministrator
{
public:
    class Delegate
    {
    public:
        virtual CHIP_ERROR GetIcacCsr(MutableByteSpan & icacCsr) { return CHIP_NO_ERROR; }
    };

    static JointFabricAdministrator & GetInstance()
    {
        static JointFabricAdministrator sInstance;
        return sInstance;
    }

    void SetPeerJFAdminClusterEndpointId(chip::EndpointId peerJFAdminClusterEndpointId);

    void SetVidVerificationForFabric(chip::FabricIndex fabricIndex);
    void ClearVidVerificationForFabric();
    bool WasVidVerificationExecutedForFabric(chip::FabricIndex fabricIndex) const;

    CHIP_ERROR SetDelegate(Delegate * delegate);
    chip::EndpointId GetPeerJFAdminClusterEndpointId() const;
    Delegate * GetDelegate();

private:
    chip::EndpointId mPeerJFAdminClusterEndpointId = chip::kInvalidEndpointId;
    std::optional<chip::FabricIndex> mVidVerificationFabricIndex;
    Delegate * mDelegate = nullptr;
};
```

The `Delegate::GetIcacCsr()` method is called during JCM when a peer
administrator sends an `ICACCSRRequest` command — the admin app generates
a CSR for the peer to sign and return via `AddICAC`.

### JCM Trust Verification

**Header**: `src/credentials/jcm/TrustVerification.h`
**Related**: `src/credentials/jcm/VendorIdVerificationClient.h`

The trust verification module implements the security validation that occurs
during the Joint Commissioning Method. From `src/credentials/jcm/TrustVerification.h`:

```cpp
struct TrustVerificationInfo
{
    EndpointId adminEndpointId   = kInvalidEndpointId;
    FabricIndex adminFabricIndex = kUndefinedFabricIndex;

    VendorId adminVendorId;
    FabricId adminFabricId;

    Platform::ScopedMemoryBufferWithSize<uint8_t> rootPublicKey;
    Platform::ScopedMemoryBufferWithSize<uint8_t> adminRCAC;
    Platform::ScopedMemoryBufferWithSize<uint8_t> adminICAC;
    Platform::ScopedMemoryBufferWithSize<uint8_t> adminNOC;
};
```

#### Trust Verification State Machine

The verification proceeds through ordered stages:

```cpp
enum TrustVerificationStage : uint8_t
{
    kIdle,
    kVerifyingAdministratorInformation,
    kPerformingVendorIDVerification,
    kAskingUserForConsent,
    kStoringEndpointID,
    kReadingCommissionerAdminFabricIndex,
    kCrossCheckingAdministratorIds,
    kComplete,
    kError,
};
```

The `TrustVerificationStateMachine` base class drives transitions:

```cpp
class TrustVerificationStateMachine
{
protected:
    virtual TrustVerificationStage GetNextTrustVerificationStage(
        const TrustVerificationStage & currentStage) = 0;
    virtual void PerformTrustVerificationStage(
        const TrustVerificationStage & nextStage) = 0;
    virtual void OnTrustVerificationComplete(TrustVerificationError error) {}

    void StartTrustVerification();
    void TrustVerificationStageFinished(
        const TrustVerificationStage & completedStage,
        const TrustVerificationError & error);

    TrustVerificationInfo mInfo;
};
```

A `TrustVerificationDelegate` receives progress callbacks and handles
user-consent prompts:

```cpp
class TrustVerificationDelegate
{
public:
    virtual void OnProgressUpdate(TrustVerificationStateMachine & sm,
                                  TrustVerificationStage stage,
                                  TrustVerificationInfo & info,
                                  TrustVerificationError error) = 0;
    virtual void OnAskUserForConsent(TrustVerificationStateMachine & sm,
                                     TrustVerificationInfo & info) = 0;
    virtual CHIP_ERROR OnLookupOperationalTrustAnchor(
        VendorId vendorID, CertificateKeyId & subjectKeyId,
        ByteSpan & globallyTrustedRootSpan) = 0;
};
```

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

#### Commands (from cluster XML)

| ID     | Command                            | Description                                  |
| ------ | ---------------------------------- | -------------------------------------------- |
| `0x00` | `AddKeySet`                        | Add a group key set                          |
| `0x01` | `UpdateKeySet`                     | Update an existing group key set             |
| `0x02` | `RemoveKeySet`                     | Remove a group key set by ID                 |
| `0x03` | `AddGroup`                         | Add a group definition                       |
| `0x04` | `UpdateGroup`                      | Update a group definition                    |
| `0x05` | `RemoveGroup`                      | Remove a group by ID                         |
| `0x06` | `AddAdmin`                         | Add an administrator entry                   |
| `0x07` | `UpdateAdmin`                      | Update an administrator entry                |
| `0x08` | `RemoveAdmin`                      | Remove an administrator entry                |
| `0x09` | `AddPendingNode`                   | Add a node in pending state                  |
| `0x0A` | `RefreshNode`                      | Pull latest state from a remote node         |
| `0x0B` | `UpdateNode`                       | Update a node's friendly name                |
| `0x0C` | `RemoveNode`                       | Remove a node from the datastore             |
| `0x0D` | `UpdateEndpointForNode`            | Update an endpoint's metadata                |
| `0x0E` | `AddGroupIDToEndpointForNode`      | Associate a group with an endpoint           |
| `0x0F` | `RemoveGroupIDFromEndpointForNode` | Remove a group from an endpoint              |
| `0x10` | `AddBindingToEndpointForNode`      | Add a binding target to an endpoint          |
| `0x11` | `RemoveBindingFromEndpointForNode` | Remove a binding from an endpoint            |
| `0x12` | `AddACLToNode`                     | Add an ACL entry to a node                   |
| `0x13` | `RemoveACLFromNode`                | Remove an ACL entry from a node              |

**Example command handler** — from `joint-fabric-datastore-server.cpp`:

```cpp
bool emberAfJointFabricDatastoreClusterAddPendingNodeCallback(
    CommandHandler * commandObj, const ConcreteCommandPath & commandPath,
    const JointFabricDatastoreCluster::Commands::AddPendingNode::DecodableType & commandData)
{
    CHIP_ERROR err                = CHIP_NO_ERROR;
    NodeId nodeId                 = commandData.nodeID;
    const CharSpan & friendlyName = commandData.friendlyName;

    SuccessOrExit(err = Server::GetInstance().GetJointFabricDatastore()
                            .AddPendingNode(nodeId, friendlyName));
exit:
    if (err == CHIP_NO_ERROR)
        commandObj->AddStatus(commandPath, Status::Success);
    else
        commandObj->AddStatus(commandPath, ClusterStatusCode(err));
    return true;
}
```

#### Attributes (from cluster XML)

| ID     | Attribute              | Type                                  |
| ------ | ---------------------- | ------------------------------------- |
| `0x00` | `AnchorRootCA`         | `octet_string`                        |
| `0x01` | `AnchorNodeID`         | `node_id`                             |
| `0x02` | `AnchorVendorID`       | `vendor_id`                           |
| `0x03` | `FriendlyName`         | `char_string` (max 32)                |
| `0x04` | `GroupKeySetList`      | `array<DatastoreGroupKeySetStruct>`   |
| `0x05` | `GroupList`            | `array<DatastoreGroupInformationEntryStruct>` |
| `0x06` | `NodeList`             | `array<DatastoreNodeInformationEntryStruct>`  |
| `0x07` | `AdminList`            | `array<DatastoreAdministratorInformationEntryStruct>` |
| `0x08` | `Status`               | `DatastoreStatusEntryStruct`          |
| `0x09` | `EndpointGroupIDList`  | `array<DatastoreEndpointGroupIDEntryStruct>`  |
| `0x0A` | `EndpointBindingList`  | `array<DatastoreEndpointBindingEntryStruct>`  |
| `0x0B` | `NodeKeySetList`       | `array<DatastoreNodeKeySetEntryStruct>`       |
| `0x0C` | `NodeACLList`          | `array<DatastoreACLEntryStruct>`              |
| `0x0D` | `NodeEndpointList`     | `array<DatastoreEndpointEntryStruct>`         |

#### Enumerations (from cluster XML)

```
DatastoreStateEnum (enum8):
  kPending      = 0x00   // Operation queued, not yet applied to device
  kCommitted    = 0x01   // Successfully applied to device
  kDeletePending = 0x02  // Deletion queued
  kCommitFailed = 0x03   // Sync to device failed

DatastoreAccessControlEntryPrivilegeEnum (enum8):
  kView = 0x01, kProxyView = 0x02, kOperate = 0x03, kManage = 0x04, kAdminister = 0x05

DatastoreAccessControlEntryAuthModeEnum (enum8):
  kPASE = 0x01, kCASE = 0x02, kGroup = 0x03
```

#### chip-tool / jf-control-app Commands for Datastore

Read datastore attributes on the admin app (node 1, endpoint 0):

```bash
# Read the anchor information
>>> jointfabricdatastore read anchor-root-ca 1 0
>>> jointfabricdatastore read anchor-node-id 1 0
>>> jointfabricdatastore read anchor-vendor-id 1 0
>>> jointfabricdatastore read friendly-name 1 0

# Read entity lists
>>> jointfabricdatastore read node-list 1 0
>>> jointfabricdatastore read admin-list 1 0
>>> jointfabricdatastore read group-list 1 0
>>> jointfabricdatastore read group-key-set-list 1 0
>>> jointfabricdatastore read status 1 0
>>> jointfabricdatastore read endpoint-group-idlist 1 0
>>> jointfabricdatastore read endpoint-binding-list 1 0
>>> jointfabricdatastore read node-key-set-list 1 0
>>> jointfabricdatastore read node-acllist 1 0
>>> jointfabricdatastore read node-endpoint-list 1 0
```

Invoke datastore commands:

```bash
# Add a pending node to the datastore
>>> jointfabricdatastore add-pending-node 1 0 --NodeID 2 --FriendlyName "light-a"

# Refresh a node (pull latest state from device)
>>> jointfabricdatastore refresh-node 1 0 --NodeID 2

# Add an admin entry
>>> jointfabricdatastore add-admin 1 0 --NodeID 100 --FriendlyName "admin-b" \
    --VendorID 0xFFF2 --ICAC hex:...

# Add a group
>>> jointfabricdatastore add-group 1 0 --GroupID 10 --FriendlyName "living-room" \
    --GroupKeySetID 1

# Remove a node
>>> jointfabricdatastore remove-node 1 0 --NodeID 2
```

### Joint Fabric Administrator Cluster (0x0753)

**Implementation**: `src/app/clusters/joint-fabric-administrator-server/`
**Cluster XML**: `src/app/zap-templates/zcl/data-model/chip/joint-fabric-administrator.xml`

This cluster handles commissioning operations and cross-admin trust
establishment.

#### Commands (from cluster XML)

| ID     | Command                            | Direction      | Description                                         |
| ------ | ---------------------------------- | -------------- | --------------------------------------------------- |
| `0x00` | `ICACCSRRequest`                   | Client→Server  | Request a CSR for issuing an ICAC                   |
| `0x01` | `ICACCSRResponse`                  | Server→Client  | Response with ICAC CSR (max 600 bytes)              |
| `0x02` | `AddICAC`                          | Client→Server  | Submit a signed ICAC (max 400 bytes)                |
| `0x03` | `ICACResponse`                     | Server→Client  | Status: OK, InvalidPublicKey, or InvalidICAC        |
| `0x04` | `OpenJointCommissioningWindow`     | Client→Server  | Open a commissioning window for a peer admin        |
| `0x05` | `TransferAnchorRequest`            | Client→Server  | Request anchor role transfer                        |
| `0x06` | `TransferAnchorResponse`           | Server→Client  | Status: OK, DatastoreBusy, or NoUserConsent         |
| `0x07` | `TransferAnchorComplete`           | Client→Server  | Finalize anchor role transfer                       |
| `0x08` | `AnnounceJointFabricAdministrator` | Client→Server  | Announce the peer admin's JFA cluster endpoint      |

#### Attributes

| ID     | Attribute                    | Type          | Description                                   |
| ------ | ---------------------------- | ------------- | --------------------------------------------- |
| `0x00` | `AdministratorFabricIndex`   | `fabric_idx` (nullable, 1-254) | The fabric this administrator operates on |

#### HandleOJCW (OpenJointCommissioningWindow)

From `joint-fabric-administrator-server.cpp` — validates parameters and opens
the window:

```cpp
void JointFabricAdministratorGlobalInstance::HandleOJCW(
    HandlerContext & ctx,
    const Commands::OpenJointCommissioningWindow::DecodableType & commandData)
{
    auto commissioningTimeout = System::Clock::Seconds16(commandData.commissioningTimeout);
    auto & pakeVerifier       = commandData.PAKEPasscodeVerifier;
    auto & discriminator      = commandData.discriminator;
    auto & iterations         = commandData.iterations;
    auto & salt               = commandData.salt;

    FabricIndex fabricIndex       = ctx.mCommandHandler.GetAccessingFabricIndex();
    const FabricInfo * fabricInfo = Server::GetInstance().GetFabricTable()
                                       .FindFabricWithIndex(fabricIndex);
    auto & failSafeContext = Server::GetInstance().GetFailSafeContext();
    auto & commissionMgr   = Server::GetInstance().GetCommissioningWindowManager();

    VerifyOrExit(fabricInfo != nullptr, ...);
    VerifyOrExit(!administratorFabricIndex.IsNull() && administratorFabricIndex.Value() != 0,
                 status.Emplace(StatusCodeEnum::kInvalidAdministratorFabricIndex));
    VerifyOrExit(failSafeContext.IsFailSafeFullyDisarmed(), status.Emplace(StatusCodeEnum::kBusy));
    VerifyOrExit(!commissionMgr.IsCommissioningWindowOpen(), status.Emplace(StatusCodeEnum::kBusy));

    // Validate PBKDF parameters
    VerifyOrExit(iterations >= kSpake2p_Min_PBKDF_Iterations, ...);
    VerifyOrExit(iterations <= kSpake2p_Max_PBKDF_Iterations, ...);
    VerifyOrExit(salt.size() >= kSpake2p_Min_PBKDF_Salt_Length, ...);

    VerifyOrExit(verifier.Deserialize(pakeVerifier) == CHIP_NO_ERROR, ...);
    VerifyOrExit(commissionMgr.OpenJointCommissioningWindow(
        commissioningTimeout, discriminator, verifier, iterations, salt,
        fabricIndex, fabricInfo->GetVendorId()) == CHIP_NO_ERROR, ...);

    ChipLogProgress(Zcl, "Commissioning window is now open");
}
```

#### HandleAnnounceJointFabricAdministrator

This command triggers JCM trust verification. The handler creates a
`JCMCommissionee` instance and starts the async verification process:

```cpp
void JointFabricAdministratorGlobalInstance::HandleAnnounceJointFabricAdministrator(
    HandlerContext & ctx,
    const Commands::AnnounceJointFabricAdministrator::DecodableType & commandData)
{
    // Clear any previous VID verification state
    Server::GetInstance().GetJointFabricAdministrator().ClearVidVerificationForFabric();

    auto onComplete = [this, cachedPath, accessingFabricIndex](const CHIP_ERROR & err) {
        if (err == CHIP_NO_ERROR)
        {
            Server::GetInstance().GetJointFabricAdministrator()
                .SetVidVerificationForFabric(accessingFabricIndex);
            commandHandler->AddStatus(cachedPath, Status::Success);
        }
        else
        {
            commandHandler->AddStatus(cachedPath, Status::Failure);
        }
        CleanupAnnounceJFA();
    };

    mActiveCommandHandle.emplace(&ctx.mCommandHandler);
    mActiveCommissionee.emplace(mActiveCommandHandle.value(),
                                 commandData.endpointID, std::move(onComplete));
    mActiveCommissionee->VerifyTrustAgainstCommissionerAdmin();
}
```

#### HandleICACCSRRequest

Validates the fail-safe is armed, VID verification was performed, then
delegates CSR generation:

```cpp
void HandleICACCSRRequest(HandlerContext & ctx, ...)
{
    // Must be invoked over CASE
    VerifyOrExit(ctx.mCommandHandler.GetSubjectDescriptor().authMode == Access::AuthMode::kCase, ...);
    VerifyOrExit(failSafeContext.IsFailSafeArmed(ctx.mCommandHandler.GetAccessingFabricIndex()), ...);

    // VID verification must have been completed first
    VerifyOrExit(jointFabricAdministrator.WasVidVerificationExecutedForFabric(
        ctx.mCommandHandler.GetAccessingFabricIndex()),
        status.Emplace(StatusCodeEnum::kVIDNotVerified));

    VerifyOrExit(jointFabricAdministrator.GetDelegate()->GetIcacCsr(icacCsr) == CHIP_NO_ERROR, ...);

    response.icaccsr = icacCsr;
    ctx.mCommandHandler.AddResponse(ctx.mRequestPath, response);
}
```

#### HandleAddICAC

Validates the ICAC against the fabric's root certificate, verifies the public
key matches the CSR, and checks DN encoding:

```cpp
void HandleAddICAC(HandlerContext & ctx, const Commands::AddICAC::DecodableType & commandData)
{
    VerifyOrExit(VerifyAddICACStep1(accessFabric, commandData) == CHIP_NO_ERROR,
                 status.Emplace(ICACResponseStatusEnum::kInvalidICAC));
    VerifyOrExit(VerifyAddICACPublicKey(commandData) == CHIP_NO_ERROR,
                 status.Emplace(ICACResponseStatusEnum::kInvalidPublicKey));
    VerifyOrExit(VerifyAddICACDNEncodingRules(commandData) == CHIP_NO_ERROR,
                 status.Emplace(ICACResponseStatusEnum::kInvalidICAC));
}

// Step 1: Validate ICAC chain against fabric Root CA
CHIP_ERROR VerifyAddICACStep1(const FabricIndex accessFabric, ...)
{
    ReturnErrorOnFailure(Server::GetInstance().GetFabricTable().FetchRootCert(accessFabric, rcacSpan));
    ReturnErrorOnFailure(certificates.LoadCert(rcacSpan, CertDecodeFlags::kIsTrustAnchor));
    ReturnErrorOnFailure(certificates.LoadCert(commandData.ICACValue, CertDecodeFlags::kGenerateTBSHash));
    validContext.mRequiredKeyUsages.Set(KeyUsageFlags::kKeyCertSign);
    validContext.mRequiredCertType = CertType::kICA;
    return certificates.ValidateCert(certificates.GetLastCert(), validContext);
}
```

#### chip-tool / jf-control-app Commands for Administrator

```bash
# Open a Joint Commissioning Window on admin app (node 11, endpoint 1)
# Parameters: node-id endpoint-id window-timeout iterations discriminator
>>> pairing open-joint-commissioning-window 11 1 400 1000 1261

# Expected log on jf-admin-app side:
# [DIS] Advertise commission parameter vendorID=65522 productID=32769
#       discriminator=1261/04 cm=3 cp=0 jf=14
```

#### JCMCommissionee

**Header**: `src/app/clusters/joint-fabric-administrator-server/JCMCommissionee.h`

The `JCMCommissionee` class inherits from both `VendorIdVerificationClient` and
`TrustVerificationStateMachine`, implementing the commissionee side of JCM:

```cpp
class JCMCommissionee : public Credentials::JCM::VendorIdVerificationClient,
                         public Credentials::JCM::TrustVerificationStateMachine
{
public:
    JCMCommissionee(CommandHandler::Handle & commandHandle,
                     EndpointId endpointId, OnCompletionFunc onCompletion);

    void VerifyTrustAgainstCommissionerAdmin();

protected:
    // VendorIdVerificationClient overrides
    CHIP_ERROR OnLookupOperationalTrustAnchor(VendorId vendorID, ...) override;
    void OnVendorIdVerificationComplete(const CHIP_ERROR & err) override;

    // TrustVerificationStateMachine overrides
    TrustVerificationStage GetNextTrustVerificationStage(...) override;
    void PerformTrustVerificationStage(...) override;
    void OnTrustVerificationComplete(TrustVerificationError error) override;

private:
    // Trust Verification Stage implementations
    TrustVerificationError StoreEndpointId();
    TrustVerificationError ReadCommissionerAdminFabricIndex();
    TrustVerificationError PerformVendorIdVerification();
    TrustVerificationError CrossCheckAdministratorIds();
    TrustVerificationError ValidateAdministratorIdsMatch(
        FabricId accessingFabricId,
        const Crypto::P256PublicKey & accessingRootPubKey) const;

    // Async attribute reads from the commissioner
    void FetchCommissionerInfo(OnCompletionFunc onComplete);
    CHIP_ERROR ReadAdminFabrics(OnCompletionFunc onComplete);
    CHIP_ERROR ReadAdminCerts(OnCompletionFunc onComplete);
    CHIP_ERROR ReadAdminNOCs(OnCompletionFunc onComplete);
};
```

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

**Commands to execute (on jf-control-app for Ecosystem A):**

```bash
# Start jf-admin-app (Ecosystem A)
$ ./jfa-app --capabilities 0x4 --passcode 11022033 --discriminator 3840 \
    --secured-device-port 5533 --rpc-server-port 33033 --KVS jfa_a_kvs

# Start jf-control-app (Ecosystem A, Vendor ID 0xFFF1)
$ ./jfc-app --rpc-server-port 33033 --storage-directory jfc_a_storage_directory \
    --commissioner-vendor-id 0xFFF1

# Commission the admin app as anchor (note: --anchor true)
>>> pairing onnetwork-long 1 11022033 3840 --anchor true

# Expected log: [JF] Anchor Administrator commissioned with success
```

Verify the NOC contains both CATs:

```bash
>>> operationalcredentials read nocs 1 0

# Decode the NOC with chip-cert:
$ chip-cert convert-cert --x509-pem <noc_bytes> - | openssl x509 -inform pem -noout -text

# Look for both CATs in Subject field:
# Subject: ..., 1.3.6.1.4.1.37244.1.6 = FFFF0001, 1.3.6.1.4.1.37244.1.6 = FFFE0001
#                                         ^^^^^^^^ Admin CAT   ^^^^^^^^ Anchor CAT
```

Commission an end device:

```bash
# Start lighting-app
$ ./chip-lighting-app --capabilities 0x4 --passcode 11022044 --KVS light_a_kvs

# Commission (controller hands off ownership to admin via RPC)
>>> pairing onnetwork 2 11022044

# Verify fabric and ACLs
>>> operationalcredentials read fabrics 2 0
>>> accesscontrol read acl 2 0

# ACL Subjects should contain Administrator CAT: FFFFFFFDFFFF0001 (hex)
```

### Stage 2: Joint Commissioning (JCM)

When a second ecosystem wants to join an existing fabric:

**Step 1 — Open Joint Commissioning Window** on Ecosystem B's admin:

```bash
# On Ecosystem B's jf-control-app:
>>> pairing open-joint-commissioning-window 11 1 400 1000 1261

# Expected log on jf-admin-app:
# [DIS] Advertise commission parameter vendorID=65522 productID=32769
#       discriminator=1261/04 cm=3 cp=0 jf=14
#
# Note: capture the manual pairing code from the log output
```

**Step 2 — Peer Admin Initiates JCM** from Ecosystem A's controller:

```bash
# On Ecosystem A's jf-control-app (note: --jcm true):
>>> pairing code 10 <manual-pairing-code> --jcm true
```

From `examples/jf-control-app/commands/pairing/PairingCommand.h`, the flags:

```cpp
AddArgument("anchor", 0, 1, &mAnchor,
    "If set to true then a NOC with Anchor and Administrator CAT is issued");
AddArgument("jcm", 0, 1, &mJCM,
    "Set it to true in order to commission a Joint Fabric Administrator");
```

These flags are mutually exclusive — using both `--anchor true` and
`--jcm true` together is an error.

**Step 3 — Trust Verification**: The `JCMCommissionee` on the target admin
performs Vendor ID verification, cross-checks administrator identities, and
requests user consent (see Trust Verification State Machine above).

**Step 4 — Finalize with AddICAC**: The peer admin submits its signed ICAC via
the `AddICAC` command. A NOC with the **Anchor CAT** (not the Administrator
CAT) is issued, granting the peer admin co-administration rights.

### Trust Verification Process

During JCM, the `JCMCommissionee` walks through these stages in order:

```
kIdle
  → kStoringEndpointID                        // Save peer's JFA endpoint
  → kReadingCommissionerAdminFabricIndex      // Read AdministratorFabricIndex
  → kVerifyingAdministratorInformation        // Fetch fabrics, certs, NOCs
  → kPerformingVendorIDVerification           // Fabric Table VID Verification
  → kCrossCheckingAdministratorIds            // Match RootPublicKey + FabricID
  → kAskingUserForConsent                     // Prompt user for approval
  → kComplete                                 // Success
```

Each stage maps to a method in `JCMCommissionee`:

1. **StoreEndpointId()** — Saves the peer admin's JFA cluster endpoint ID.
2. **ReadCommissionerAdminFabricIndex()** — Reads `AdministratorFabricIndex`
   attribute from the peer's JFA cluster.
3. **FetchCommissionerInfo()** — Reads the peer's Fabrics, TrustedRootCertificates,
   and NOCs attributes from their OperationalCredentials cluster.
4. **PerformVendorIdVerification()** — Runs the Fabric Table Vendor ID
   Verification Procedure against the fabric indicated by
   `AdministratorFabricIndex`.
5. **CrossCheckAdministratorIds()** — Verifies that the RootPublicKey and
   FabricID of the accessing fabric match the Fabric indicated by
   `AdministratorFabricIndex`.
6. **User Consent** — The `TrustVerificationDelegate::OnAskUserForConsent()` callback
   is invoked. The flow pauses until `ContinueAfterUserConsent(true/false)` is called.

---

## CASE Authenticated Tags (CATs)

Joint Fabric uses two types of CATs embedded in NOC certificates to control
privilege levels.

From `src/lib/core/CASEAuthTag.h`:

```cpp
static constexpr uint16_t kAdminCATIdentifier  = 0xFFFF;
static constexpr uint16_t kAnchorCATIdentifier = 0xFFFE;

constexpr CASEAuthTag GetAdminCATWithVersion(uint16_t version)
{
    return ((static_cast<uint32_t>(kAdminCATIdentifier) << 16) | version);
}

constexpr CASEAuthTag GetAnchorCATWithVersion(uint16_t version)
{
    return ((static_cast<uint32_t>(kAnchorCATIdentifier) << 16) | version);
}
```

Initial versions are configured in
`examples/jf-control-app/include/CHIPProjectAppConfig.h`:

```cpp
#define CHIP_CONFIG_ADMINISTRATOR_CAT_INITIAL_VERSION 0x0001
#define CHIP_CONFIG_ANCHOR_CAT_INITIAL_VERSION 0x0001
```

| CAT Type            | Pattern      | Example      | Description                           |
| ------------------- | ------------ | ------------ | ------------------------------------- |
| **Administrator CAT** | `0xFFFF00xx` | `0xFFFF0001` | Identifies an administrator on the fabric. The anchor admin receives this in addition to the Anchor CAT. |
| **Anchor CAT**      | `0xFFFE00xx` | `0xFFFE0001` | Identifies the anchor administrator. All admins on the fabric receive this CAT. |

### CAT Usage in NOC Issuance

- **Anchor commissioning** (`--anchor true`): NOC contains both Administrator
  CAT and Anchor CAT. `CaseAdminSubject` is set to the Administrator CAT.
- **JCM commissioning** (`--jcm true`): NOC contains Administrator CAT.
  `CaseAdminSubject` is set to the Anchor CAT.
- **Device commissioning**: Standard NOC without Joint Fabric CATs. ACL subjects
  reference the Administrator CAT.

In the NOC's X.509 Subject field, CATs appear as:

```
Subject: ..., 1.3.6.1.4.1.37244.1.6 = FFFF0001, 1.3.6.1.4.1.37244.1.6 = FFFE0001
```

### CAT Validation in AddNOC

During JCM, the Operational Credentials cluster performs extra validation.
From `src/app/clusters/operational-credentials-server/OperationalCredentialsCluster.cpp`:

```cpp
#if CHIP_DEVICE_CONFIG_ENABLE_JOINT_FABRIC
    // These checks should only run during JCM.
    if (commissioningWindowManager.IsJCM())
    {
        // NOC must contain an Administrator CAT (0xFFFF)
        CATValues cats;
        err = ExtractCATsFromOpCert(NOCValue, cats);
        VerifyOrExit(err == CHIP_NO_ERROR && cats.ContainsIdentifier(kAdminCATIdentifier),
                     nocResponse = NodeOperationalCertStatusEnum::kInvalidNOC);

        // CaseAdminSubject must contain an Anchor CAT (0xFFFE)
        CASEAuthTag tag = CASEAuthTagFromNodeId(commandData.caseAdminSubject);
        VerifyOrExit(IsCASEAuthTag(commandData.caseAdminSubject) && IsValidCASEAuthTag(tag) &&
                         (GetCASEAuthTagIdentifier(tag) == kAnchorCATIdentifier),
                     nocResponse = NodeOperationalCertStatusEnum::kInvalidAdminSubject);
    }
#endif // CHIP_DEVICE_CONFIG_ENABLE_JOINT_FABRIC
```

The `CATValues` struct provides helper methods for checking tags:

```cpp
// Returns true if this set contains any version of the identifier
bool ContainsIdentifier(uint16_t identifier) const;

// Returns true if subject matches one of the CATs (version-aware)
bool CheckSubjectAgainstCATs(NodeId subject) const;
```

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

When `RefreshNode()` is called, the datastore walks through an async state
machine defined by `ContinueRefresh()`. From
`src/app/server/JointFabricDatastore.h`:

```cpp
enum RefreshState
{
    kIdle,
    kRefreshingEndpoints,
    kRefreshingGroups,
    kRefreshingBindings,
    kFetchingGroupKeySets,
    kRefreshingGroupKeySets,
    kRefreshingACLs,
};
```

Flow:

```
kIdle  ──► FetchEndpointList()
           ──► kRefreshingEndpoints   (add/remove endpoint entries)
               ──► kRefreshingGroups  (for each endpoint: FetchEndpointGroupList())
                   ──► sync pending/delete-pending group entries
                       ──► kRefreshingBindings (for each endpoint: FetchEndpointBindingList())
                           ──► sync all bindings via SyncNode()
                               ──► kFetchingGroupKeySets (FetchGroupKeySetList())
                                   ──► kRefreshingGroupKeySets (sync each key set)
                                       ──► kRefreshingACLs (FetchACLList())
                                           ──► sync ACL entries
                                               ──► Mark node as Committed
                                                   ──► kIdle
```

Each stage invokes a `Delegate::Fetch*()` or `Delegate::SyncNode()` call with
an async callback. On success, the callback updates the datastore state and
calls `ContinueRefresh()` to advance. On failure, the state machine resets to
`kIdle` and the node remains in `Pending` state.

**Example from `ContinueRefresh()` — the kIdle → kRefreshingEndpoints transition:**

```cpp
case kIdle: {
    ReturnErrorOnFailure(SetNode(mRefreshingNodeId,
        Clusters::JointFabricDatastore::DatastoreStateEnum::kPending));

    ReturnErrorOnFailure(mDelegate->FetchEndpointList(mRefreshingNodeId,
        [this](CHIP_ERROR err,
               const std::vector<DatastoreEndpointEntryStruct::Type> & endpoints) {
            if (err == CHIP_NO_ERROR)
            {
                mRefreshingEndpointsList = endpoints;
                mRefreshState = kRefreshingEndpoints;
            }
            else
            {
                mRefreshingNodeId = kUndefinedNodeId;
                mRefreshState     = kIdle;
                return;
            }
            ContinueRefresh();  // advance to kRefreshingEndpoints
        }));
}
```

**Handling pending/delete-pending entries during group refresh:**

```cpp
if (it->statusEntry.state == DatastoreStateEnum::kPending)
{
    auto entryToSync = *it;
    mDelegate->SyncNode(mRefreshingNodeId, entryToSync, [this, idx]() {
        mEndpointGroupIDEntries[idx].statusEntry.state = DatastoreStateEnum::kCommitted;
    });
}
else if (it->statusEntry.state == DatastoreStateEnum::kDeletePending)
{
    mDelegate->SyncNode(mRefreshingNodeId, nullEntry, [this, entryToErase]() {
        // Erase the entry after successful deletion on device
        mEndpointGroupIDEntries.erase(...);
    });
}
```

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

**Initialization** — from `examples/jf-admin-app/linux/main.cpp`:

```cpp
void ApplicationInit()
{
    SuccessOrDie(JFAMgr().Init(Server::GetInstance()));
    SuccessOrDie(JFADSync().Init(Server::GetInstance()));

    // Wire up delegates
    Server::GetInstance().GetJointFabricAdministrator().SetDelegate(&JFAMgr());
    Server::GetInstance().GetJointFabricDatastore().SetDelegate(&JFADSync());
}
```

### RPC Protocol Between Controller and Admin

The jf-control-app communicates with jf-admin-app via Pigweed RPC.
From `examples/common/pigweed/protos/joint_fabric_service.proto`:

```protobuf
message OwnershipContext {
  uint64 node_id = 1;           // node to finalize commissioning with
  bool jcm = 2;                 // true if JCM process should be finalized
  bytes trustedIcacPublicKeyB = 3;
  uint32 peerAdminJFAdminClusterEndpointId = 4;
}

enum TransactionType {
  ICAC_CSR = 0;
  CROSS_SIGNED_ICAC = 1;
}

message RequestOptions {
  TransactionType transaction_type = 1;
  uint64 anchor_fabric_id = 2;   // required only for CROSS_SIGNED_ICAC
}

message Response {
  TransactionType transaction_type = 1;
  bytes response_bytes = 2;
}

service JointFabric {
  rpc TransferOwnership(OwnershipContext) returns (pw.protobuf.Empty) {}
  rpc GetStream(pw.protobuf.Empty) returns (stream RequestOptions);
  rpc ResponseStream(Response) returns (pw.protobuf.Empty);
}
```

After commissioning a device, the controller calls `TransferOwnership` to hand
off node management to the admin app. During JCM, the `GetStream` and
`ResponseStream` RPCs exchange ICAC CSR and cross-signed ICAC data.

---

## Unit Testing

Enable Joint Fabric tests with:

```bash
gn gen out/test --args="chip_device_config_enable_joint_fabric=true"
ninja -C out/test TestJointFabricDatastore
./out/test/tests/TestJointFabricDatastore
```

From `src/app/server/tests/TestJointFabricDatastore.cpp` — usage examples:

```cpp
// Add a pending node and verify listener notification
TEST(JointFabricDatastoreTest, AddPendingNodeNotifiesListener)
{
    JointFabricDatastore store;
    DummyListener listener;
    store.AddListener(listener);

    CHIP_ERROR err = store.AddPendingNode(123, CharSpan::fromCharString("controller-a"));
    EXPECT_EQ(err, CHIP_NO_ERROR);
    EXPECT_TRUE(listener.mNotified);
}

// Refresh a node (triggers async state machine through DummyDelegate)
TEST(JointFabricDatastoreTest, RefreshNodeUpdatesExistingNode)
{
    JointFabricDatastore store;
    DummyDelegate delegate;
    store.SetDelegate(&delegate);

    store.AddPendingNode(123, CharSpan::fromCharString("controller-a"));
    CHIP_ERROR err = store.RefreshNode(123);
    EXPECT_EQ(err, CHIP_NO_ERROR);
}

// Add and remove an administrator
TEST(JointFabricDatastoreTest, AddAndRemoveAdmin)
{
    JointFabricDatastore store;
    DummyDelegate delegate;
    store.SetDelegate(&delegate);

    DatastoreAdministratorInformationEntryStruct::Type admin;
    admin.nodeID = 100;
    EXPECT_EQ(store.AddAdmin(admin), CHIP_NO_ERROR);
    EXPECT_EQ(store.RemoveAdmin(100), CHIP_NO_ERROR);
}

// Add group, endpoint, group-to-endpoint association, then remove
TEST(JointFabricDatastoreTest, RemoveGroupIDFromEndpoint)
{
    JointFabricDatastore store;
    DummyDelegate delegate;
    store.SetDelegate(&delegate);

    store.TestAddNodeKeySetEntry(/*groupId=*/10, /*keySetId=*/1, /*nodeId=*/123);
    store.TestAddEndpointEntry(/*endpointId=*/1, /*nodeId=*/123, CharSpan::fromCharString("ep"));

    Commands::AddGroup::DecodableType addGroupData;
    addGroupData.groupID       = 10;
    addGroupData.friendlyName  = CharSpan::fromCharString("test-group");
    addGroupData.groupKeySetID = 1;
    store.AddGroup(addGroupData);

    store.AddPendingNode(123, CharSpan::fromCharString("test"));
    store.AddGroupIDToEndpointForNode(123, 1, 10);
    EXPECT_EQ(store.RemoveGroupIDFromEndpointForNode(123, 1, 10), CHIP_NO_ERROR);
}
```

The test `DummyDelegate` immediately invokes success callbacks, allowing
synchronous testing of the async state machine:

```cpp
class DummyDelegate : public JointFabricDatastore::Delegate
{
public:
    CHIP_ERROR SyncNode(NodeId nodeId,
        const DatastoreEndpointGroupIDEntryStruct::Type & entry,
        std::function<void()> onSuccess) override
    {
        onSuccess();  // Immediately succeed
        return CHIP_NO_ERROR;
    }
    // ... similar for all other SyncNode/Fetch* overloads ...
};
```

---

## Datastore Limits

From `src/app/server/JointFabricDatastore.h`:

```cpp
static constexpr size_t kMaxNodes            = 256;
static constexpr size_t kMaxAdminNodes       = 32;
static constexpr size_t kMaxGroups           = kMaxNodes / 16;  // = 16
static constexpr size_t kMaxGroupKeySet      = kMaxGroups * 16; // = 256
static constexpr size_t kMaxFriendlyNameSize = 32;
static constexpr size_t kMaxACLs             = 64;
```

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
