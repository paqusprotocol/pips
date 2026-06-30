# PIP-0002: XPQ Ethereum Mainnet Bridge

## Status

Draft

## Summary

Introduce a bridge between Paqus Layer 1 and Ethereum mainnet so native XPQ can be represented on Ethereum as a wrapped ERC-20 token, `wXPQ`. Users lock XPQ on Paqus L1 and mint the equivalent amount of `wXPQ` on Ethereum. Users burn `wXPQ` on Ethereum and unlock the equivalent XPQ on Paqus L1.

The bridge should preserve a strict 1:1 backing model:

```text
1 XPQ locked on Paqus L1 == 1 wXPQ minted on Ethereum mainnet
```

## Motivation

XPQ is native to Paqus L1, while Ethereum mainnet has the deepest existing liquidity, wallet support, custody tooling, and DeFi integrations. A wrapped XPQ token gives XPQ holders access to Ethereum applications without changing Paqus consensus or making Ethereum part of Paqus block validation.

The first version should solve four problems:

- Allow XPQ holders to move value from Paqus L1 to Ethereum mainnet.
- Allow Ethereum users and applications to hold and transfer wrapped XPQ as an ERC-20 token.
- Keep bridge accounting auditable and 1:1 backed.
- Provide a foundation for future liquidity, exchange, and DeFi integrations.

## Non-Goals

- The bridge does not make Ethereum a Paqus consensus dependency.
- The bridge does not validate Ethereum state inside Paqus consensus in the first version.
- The bridge does not support arbitrary message passing.
- The bridge does not bridge assets from Ethereum into Paqus other than returning wrapped XPQ back to native XPQ.
- The bridge does not define liquidity pools, exchange listings, or DeFi incentives.
- The bridge does not change XPQ supply rules.

## Terminology

- `XPQ`: Native Paqus Layer 1 coin.
- `wXPQ`: ERC-20 wrapped XPQ deployed on Ethereum mainnet.
- `Lock`: A Paqus L1 transaction that deposits XPQ into the bridge custody address.
- `Mint`: An Ethereum transaction that creates `wXPQ` after a Paqus lock is confirmed.
- `Burn`: An Ethereum transaction that destroys `wXPQ` and requests native XPQ release.
- `Unlock`: A Paqus L1 transaction that releases XPQ after an Ethereum burn is finalized.
- `Bridge operator`: An off-chain service that observes both chains and submits bridge transactions.
- `Validator set`: The authorized bridge signers for the first version.

## Architecture

```text
Paqus L1                                  Ethereum mainnet

user XPQ
   |
   v
bridge lock address -- observed by --> bridge operators
                                      |
                                      v
                              wXPQ ERC-20 contract


wXPQ ERC-20 burn -- observed by --> bridge operators
                                      |
                                      v
                              Paqus L1 unlock transaction
```

The first bridge version should use an explicit lock/mint and burn/unlock design:

1. User sends XPQ to the Paqus bridge custody address with Ethereum recipient metadata.
2. Bridge operators wait for Paqus finality.
3. Operators submit an Ethereum mint authorization to the `wXPQ` bridge contract.
4. The Ethereum contract mints `wXPQ` to the requested Ethereum address.
5. To return, user burns `wXPQ` with a Paqus recipient address.
6. Bridge operators wait for Ethereum finality.
7. Operators submit a Paqus unlock transaction to release native XPQ.

## Trust Model

The draft first version is a federated bridge. A threshold of authorized bridge signers controls minting on Ethereum and unlocking on Paqus.

Initial policy:

- Minting requires threshold signatures from the bridge validator set.
- Unlocking requires threshold authorization from the same validator set.
- Validator-set changes require an on-chain governance or multisig-controlled upgrade process.
- All lock, mint, burn, and unlock events must be publicly auditable.

Future versions may replace or reduce federation trust with light-client verification, zero-knowledge proofs, or other trust-minimized mechanisms.

## Paqus L1 Requirements

Paqus must support a canonical bridge lock format that is easy for operators and auditors to parse.

Minimum fields:

- `amount`
- `paqus_sender`
- `ethereum_recipient`
- `nonce` or unique transaction id
- `bridge_id`
- optional `memo`

The bridge custody address should be fixed per network and documented before launch. XPQ locked in this address backs the circulating `wXPQ` supply on Ethereum.

Paqus lock transactions must only become mintable after Paqus finality. The first version should use the existing finality depth defined by the protocol.

## Ethereum Contract Requirements

Deploy an ERC-20 token contract for `wXPQ` on Ethereum mainnet.

Required behavior:

- ERC-20 compatible transfers, approvals, and allowances.
- Minting only through the bridge mint function.
- Burning through a bridge burn function that records the target Paqus recipient.
- Replay protection for processed Paqus lock events.
- Replay protection for processed Ethereum burn events.
- Pausable bridge operations for emergencies.
- Public events for `MintedFromPaqus` and `BurnedForPaqus`.

Suggested token metadata:

```text
name: Wrapped XPQ
symbol: wXPQ
decimals: match native XPQ display precision
```

## Bridge Operator Responsibilities

- Observe finalized Paqus lock transactions.
- Observe finalized Ethereum burn events.
- Verify bridge metadata and recipient addresses.
- Produce threshold authorizations for mint and unlock operations.
- Submit Ethereum transactions for minting `wXPQ`.
- Submit Paqus transactions for unlocking XPQ.
- Maintain an auditable database of processed events.
- Refuse duplicate, malformed, expired, or unsupported bridge requests.

## Finality Policy

Initial policy:

- Paqus-to-Ethereum minting: wait for Paqus finality before mint authorization.
- Ethereum-to-Paqus unlocking: wait for a conservative Ethereum mainnet confirmation window before unlock authorization.

The Ethereum confirmation count should be configurable and documented at deployment time.

## Supply Invariant

The bridge must maintain this invariant:

```text
wXPQ totalSupply <= XPQ locked in the Paqus bridge custody address
```

Auditing tools should compare:

- Paqus bridge custody balance.
- Ethereum `wXPQ` total supply.
- Pending finalized locks not yet minted.
- Pending finalized burns not yet unlocked.
- Failed or manually recovered bridge operations.

## Security Considerations

Minimum first-version controls:

- Threshold signer authorization for mint and unlock operations.
- Replay protection on both chains.
- Strict amount and recipient validation.
- Emergency pause on Ethereum mint and burn paths.
- Monitoring for supply invariant violations.
- Public bridge event logs.
- Rate limits or per-transaction caps during early launch.
- Independent contract audit before Ethereum mainnet deployment.

Future controls:

- Timelocked bridge upgrades.
- Signer rotation with public announcements.
- Slashing or bond requirements for operators.
- Trust-minimized verification of Paqus finality on Ethereum.
- Trust-minimized verification of Ethereum finality for Paqus unlocks.

## Open Questions

- What exact decimal precision should `wXPQ` use?
- Should the first Paqus lock format be a dedicated transaction type or a standard transfer with structured memo data?
- What threshold and validator-set size should launch with mainnet bridge operations?
- How many Ethereum confirmations should be required before unlocking XPQ?
- Should bridge fees be charged in XPQ, ETH, or both?
- What governance process controls contract upgrades and signer rotation?
- Should the bridge launch on Ethereum testnet before mainnet, and which testnet should be used?

## Rollout Plan

1. Define the canonical Paqus bridge lock format and custody address rules.
2. Implement bridge event indexing for Paqus L1.
3. Implement and test the `wXPQ` ERC-20 bridge contract.
4. Implement bridge operator services for lock/mint and burn/unlock flows.
5. Add supply-invariant auditing tools.
6. Run an Ethereum testnet deployment with Paqus testnet XPQ.
7. Complete independent security review.
8. Deploy `wXPQ` to Ethereum mainnet.
9. Launch mainnet bridge with conservative limits and monitoring.
