# Teku Peer Searching Algorithm

## Overview

Teku uses a sophisticated peer searching and selection algorithm to maintain optimal connectivity in the Ethereum beacon chain network. The algorithm combines peer discovery using the DiscV5 protocol with intelligent peer selection strategies that balance random peer selection with score-based selection to ensure network diversity and subnet coverage.

## Core Components

### 1. Discovery Layer (DiscV5Service)

Teku uses the **DiscV5 (Discovery v5)** protocol for peer discovery, which is the standard peer discovery protocol for Ethereum 2.0 clients.

#### Key Features:
- **Bootstrap Nodes**: Connects to configured bootstrap nodes to join the network
- **Periodic Bootnode Refresh**: Pings bootnodes every 2 minutes to maintain connections
- **Live Node Tracking**: Maintains a real-time view of active peers in the network
- **ENR (Ethereum Node Records)**: Uses ENR for peer information exchange, including:
  - Node ID and public key
  - IP addresses (IPv4/IPv6 dual-stack support)
  - UDP/TCP ports
  - Custom fields (e.g., attestation subnets, sync committee subnets)

#### Search Process:
The `searchForPeers()` method triggers the underlying DiscV5 discovery system to actively search for new peers in the network using the Kademlia-based DHT (Distributed Hash Table) approach.

### 2. Connection Management (ConnectionManager)

The `ConnectionManager` orchestrates the peer searching and connection process with two distinct operational modes:

#### Warmup Mode
- **Trigger**: When the node has zero connected peers
- **Interval**: Searches for peers every **1 second**
- **Purpose**: Quickly establish initial connections to join the network

#### Normal Operation Mode
- **Trigger**: When the node has at least one connected peer
- **Interval**: Searches for peers every **30 seconds**
- **Purpose**: Maintain optimal peer count and network health

#### Connection Workflow:
1. Trigger discovery search via `DiscoveryService.searchForPeers()`
2. Combine discovered peers with already known peers
3. Filter peers based on:
   - Not on any local address
   - Pass all configured peer predicates
   - Not already connected
   - Reputation allows connection
4. Use `PeerSelectionStrategy` to select best peers to connect
5. Attempt connections to selected peers
6. Track connection attempts with metrics (attempted, successful, failed)

#### Static Peers:
- Configured static peers are maintained with **persistent connections**
- Automatic reconnection after **20 seconds** if disconnected
- Always prioritized for connection regardless of peer limits

### 3. Peer Selection Strategy (Eth2PeerSelectionStrategy)

The peer selection strategy uses a **two-pool approach** to balance network diversity with targeted subnet coverage:

#### Pool 1: Randomly Selected Peers
- **Purpose**: Ensure network diversity and prevent echo chambers
- **Configuration**: Minimum number of randomly selected peers (configurable via `minimumRandomlySelectedPeerCount`)
- **Selection**: Random shuffle of candidate peers
- **Priority**: Selected first before score-based peers

#### Pool 2: Score-Based Peers
- **Purpose**: Ensure adequate subnet coverage for attestations and sync committees
- **Selection**: Peers sorted by score (highest first)
- **Scoring**: Based on subnet subscriptions (see Peer Scoring section)

#### Selection Algorithm:

```
1. Calculate peers needed:
   - peersRequiredForPeerCount = (currentPeerCount < lowerBound) ? (upperBound - currentPeerCount) : 0
     Note: When below lower bound, we add enough peers to reach the upper bound in one operation
   - randomlySelectedPeersToAdd = max(0, minimumRandomlySelected - currentRandomlySelected)
   - peersRequiredForSubnets = targetSubnetSubscriberCount - minCurrentSubscribersForAnyRelevantSubnet
     (Ensures we have enough peers covering all subnets we care about)
   - scoreBasedPeersToAdd = max(peersRequiredForPeerCount - currentRandomlySelected, peersRequiredForSubnets)

2. Select randomly selected peers:
   - Shuffle all candidate peers
   - Pick up to randomlySelectedPeersToAdd candidates
   - Mark them as RANDOMLY_SELECTED in peer pools

3. Select score-based peers:
   - Score all remaining candidates based on subnet subscriptions
   - Sort by score (descending)
   - Pick top scoreBasedPeersToAdd candidates
   - Mark them as SCORE_BASED in peer pools

4. Return combined list of selected peers
```

### 4. Peer Scoring (SubnetScorer)

Peers are scored based on their **subnet subscriptions** to ensure the node has adequate coverage for:
- **Attestation subnets** (64 subnets)
- **Sync committee subnets** (4 subnets)

#### Scoring Algorithm:

The algorithm prioritizes peers that subscribe to subnets with fewer existing subscribers:

```
For each subnet the peer subscribes to:
  score += MAX_SUBNET_SCORE / ((numberOfOtherSubscribers + 1)²)

Total peer score = attestationSubnetScore + syncCommitteeSubnetScore
```

Where:
- `MAX_SUBNET_SCORE = 1000`
- `numberOfOtherSubscribers` = count of currently connected peers subscribed to that subnet

#### Key Properties:
- **High scores** for peers on underrepresented subnets (fewer subscribers)
- **Low scores** for peers on well-covered subnets (many subscribers)
- **Quadratic penalty** for additional subscribers (inverse square relationship)
- Only scores subnets that are **relevant** to the local node's current needs

#### Scoring Examples:
- Subnet with 0 other subscribers: 1000 / (1²) = **1000 points**
- Subnet with 1 other subscriber: 1000 / (2²) = **250 points**
- Subnet with 2 other subscribers: 1000 / (3²) = **111 points**
- Subnet with 9 other subscribers: 1000 / (10²) = **10 points**

### 5. Target Peer Range (TargetPeerRange)

Manages the acceptable range of peer connections:

#### Configuration Parameters:
- **lowerBound**: Minimum desired peer count
- **upperBound**: Maximum desired peer count
- **minimumRandomlySelectedPeerCount**: Minimum peers from random selection

#### Behavior:
- **Adding Peers**: When `currentPeerCount < lowerBound`, add enough peers to reach `upperBound`
- **Dropping Peers**: When `currentPeerCount > upperBound`, drop excess peers
- **Hysteresis**: The gap between lower and upper bounds prevents thrashing

### 6. Peer Pool Management (PeerPools)

Tracks peer connection types to maintain the two-pool strategy:

#### Connection Types:
- **STATIC**: Manually configured persistent peers
- **RANDOMLY_SELECTED**: Peers chosen via random selection for diversity
- **SCORE_BASED**: Peers chosen based on subnet scoring (default pool)

#### Pool Operations:
- Peers start in SCORE_BASED pool by default
- Can be moved to RANDOMLY_SELECTED pool during selection
- RANDOMLY_SELECTED peers can be demoted to SCORE_BASED during disconnection selection
- STATIC peers are never dropped automatically

### 7. Peer Disconnection Strategy

When the node has too many peers (`currentPeerCount > upperBound`):

#### Disconnection Algorithm:

```
1. Calculate peers to drop:
   peersToDrop = currentPeerCount - upperBound

2. Calculate randomly selected peers to demote:
   randomPeersToDrop = min(
     currentRandomlySelected - minimumRandomlySelected,
     peersToDrop
   )

3. Demote random peers:
   - Shuffle randomly selected peers
   - Demote first randomPeersToDrop to SCORE_BASED pool

4. Disconnect lowest scoring peers:
   - Score all peers in SCORE_BASED pool (including demoted)
   - Sort by score (ascending - lowest first)
   - Disconnect first peersToDrop peers
   - Reason: TOO_MANY_PEERS
```

This ensures:
- Randomly selected peers are preserved up to the minimum threshold
- Lowest-value peers (least useful subnet coverage) are dropped first
- Network diversity is maintained even during disconnections

## Configuration and Tuning

### Key Configuration Points:

1. **Target Peer Range**:
   - Lower bound: Minimum peers before triggering peer search
   - Upper bound: Maximum peers before triggering disconnections
   - Recommended gap: Prevents connection/disconnection thrashing

2. **Minimum Randomly Selected Peers**:
   - Ensures network diversity
   - Prevents relying solely on high-scoring peers
   - Typical value: 20-30% of total peer count

3. **Discovery Intervals**:
   - Warmup: 1 second (fast initial connection)
   - Normal: 30 seconds (steady-state maintenance)
   - Bootnode refresh: 2 minutes

4. **Static Peer Reconnection**:
   - Reconnection delay: 20 seconds after disconnection
   - Ensures persistent connections to trusted peers

## Algorithm Flow Summary

```
┌─────────────────────────────────────────────────────────────┐
│                    Connection Manager                        │
│                      (Every 30 seconds)                      │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
           ┌────────────────────────────────┐
           │  DiscV5 Discovery Service      │
           │  searchForPeers()              │
           └───────────┬────────────────────┘
                       │
                       │ Discovered Peers
                       ▼
           ┌────────────────────────────────┐
           │  Combine with Known Peers      │
           │  + Filter (valid, not          │
           │    connected, reputation OK)   │
           └───────────┬────────────────────┘
                       │
                       │ Candidate Peers
                       ▼
           ┌────────────────────────────────┐
           │  Eth2 Peer Selection           │
           │  Strategy                      │
           │                                │
           │  1. Select Random Peers        │
           │     (shuffle + pick)           │
           │                                │
           │  2. Score Remaining Peers      │
           │     (subnet coverage)          │
           │                                │
           │  3. Select Top Scored Peers    │
           └───────────┬────────────────────┘
                       │
                       │ Selected Peers
                       ▼
           ┌────────────────────────────────┐
           │  Attempt Connections           │
           │  + Track Metrics               │
           │  + Subscribe to Disconnects    │
           └────────────────────────────────┘
```

## Benefits of This Algorithm

1. **Network Diversity**: Random peer selection prevents clustering and echo chambers
2. **Subnet Coverage**: Score-based selection ensures adequate coverage for all relevant subnets
3. **Adaptive**: Automatically adjusts to subnet subscription needs
4. **Resilient**: Fast warmup mode ensures quick network joining
5. **Efficient**: Quadratic scoring prioritizes underrepresented subnets
6. **Stable**: Hysteresis in peer range prevents connection thrashing
7. **IPv6 Ready**: Supports dual-stack (IPv4/IPv6) networking
8. **Metrics-Driven**: Comprehensive connection attempt tracking

## Related Files

### Core Implementation:
- `networking/p2p/src/main/java/tech/pegasys/teku/networking/p2p/discovery/DiscoveryService.java`
- `networking/p2p/src/main/java/tech/pegasys/teku/networking/p2p/discovery/discv5/DiscV5Service.java`
- `networking/p2p/src/main/java/tech/pegasys/teku/networking/p2p/connection/ConnectionManager.java`
- `networking/eth2/src/main/java/tech/pegasys/teku/networking/eth2/peers/Eth2PeerSelectionStrategy.java`
- `networking/eth2/src/main/java/tech/pegasys/teku/networking/eth2/gossip/subnets/SubnetScorer.java`

### Supporting Classes:
- `networking/p2p/src/main/java/tech/pegasys/teku/networking/p2p/connection/TargetPeerRange.java`
- `networking/p2p/src/main/java/tech/pegasys/teku/networking/p2p/connection/PeerPools.java`
- `networking/p2p/src/main/java/tech/pegasys/teku/networking/p2p/connection/PeerSelectionStrategy.java`
- `networking/eth2/src/main/java/tech/pegasys/teku/networking/eth2/peers/PeerScorer.java`
