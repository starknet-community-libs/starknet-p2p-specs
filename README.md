# StarkNet P2P Specification

This repo contains and tracks the specification of the P2P protocol for StarkNet nodes.

There are three different networks that serve different type of applications:
* [Sync](./p2p/proto/sync/protocols.md) - responsible for downloading information about blocks existing in Starknet
* [Mempool](./p2p/proto/mempool/mempool.md) - responsible for handling transactions pending for insertion to Starknet
* Consensus - responsible for creating and adding new blocks to Starknet via staking
