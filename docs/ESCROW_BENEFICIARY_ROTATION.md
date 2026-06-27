# Beneficiary Rotation System

This document describes the beneficiary rotation system implemented in the LiquiFact Escrow contract.

## Overview

The beneficiary rotation system allows the current SME (Small Medium Enterprise) and the admin to jointly update the SME beneficiary address (`sme_address`). This is useful for assigning the rights to a different entity (e.g., factoring) without needing to redeploy the escrow contract.

## Key Features

- **Dual Consent Authorization**: Requires authorization from both the current SME address and the admin address.
- **State Guard**: Only allowed when the escrow is in a non-terminal state (open or funded).
- **Collateral Ownership**: Metadata ownership intrinsically transfers with the `sme_address`.
- **Event Transparency**: A `BeneficiaryRotated` event is emitted upon successful rotation.

## Core Functions

### `rotate_beneficiary`

Updates the SME beneficiary address.

**Parameters:**

- `new_sme`: The address that will become the new SME.

**Requirements:**

- Escrow status must be `0` (open) or `1` (funded).
- Authorization required from **current SME address**.
- Authorization required from **admin address**.
- The `new_sme` address cannot be the same as the current SME.

**Events:**

- `BeneficiaryRotated` - Emitted when the rotation is successful, detailing the `old_sme` and `new_sme`.

## Integration with Existing Functions

### `withdraw` and `settle`

These functions rely on `sme_address.require_auth()`. After `rotate_beneficiary` is called, the new address becomes the sole authority for these functions. 

### `record_sme_collateral_commitment`

Collateral metadata commitments are tied to the active SME. When the beneficiary is rotated, the new SME automatically gains the right to update these commitments.

## Security Considerations

### Dual Consent

Both the admin and the current SME must sign the transaction. The admin alone cannot forcibly reassign the beneficiary, protecting the SME from malicious admin actions. The SME alone cannot transfer the rights, maintaining governance oversight.

### Terminal States

Rotation is blocked if the escrow is settled, withdrawn, or cancelled. This prevents unexpected state changes during or after the distribution of funds.
