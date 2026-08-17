# Technical notes — ed25519 verification bypass

## How the check is supposed to work

Solana programs can't verify an ed25519 signature cheaply inside their
own code. The standard pattern: a native `Ed25519Program` instruction
sits somewhere in the same transaction and does the actual crypto
check (transaction fails outright if that signature is invalid). The
calling program's only job is to confirm that instruction is really
there and says what's expected.

## Where it breaks

`utils/ed25519.rs::verify_ed25519_ix`, called from `pre_transfer`
(`instructions/transfer/common.rs`), which both `transfer_ix` and
`transfer_v2_ix` depend on.

```rust
let ed25519_offsets = Ed25519SignatureOffsets {
    signature_offset: ...,
    signature_instruction_index: u16::from_le_bytes([data[4], data[5]]),
    public_key_offset: ...,
    public_key_instruction_index: u16::from_le_bytes([data[8], data[9]]),
    message_data_offset: ...,
    message_data_size: ...,
    message_instruction_index: u16::from_le_bytes([data[14], data[15]]),
};
...
// Validate that all instruction indices are the same
if ed25519_offsets.signature_instruction_index != ed25519_offsets.public_key_instruction_index
    || ed25519_offsets.signature_instruction_index != ed25519_offsets.message_instruction_index
{
    return Err(...);
}
```

The `*_instruction_index` fields tell the runtime which instruction in
the transaction to actually pull signature/pubkey/message bytes from
for the crypto check. Standard convention (what the SDK's
`new_ed25519_instruction` helper produces) is `u16::MAX` — "use this
same instruction." The code above checks the three indices match each
other. It never checks they equal `u16::MAX`, or the index of the
ed25519 instruction itself.

That's the whole bug. It breaks the link between what the native
program actually verified cryptographically and what this code reads
and compares:

```rust
let pubkey = Pubkey::try_from(&verify_instruction.data[pk_offset..pk_offset+32])...;
if pubkey.to_bytes() != pub_key { return Err(...); }
let message_data = &verify_instruction.data[msg_offset..msg_offset+msg_size];
if message_data != msg { return Err(...); }
```

`verify_instruction` here is always "the instruction at position -1,"
read directly — regardless of what the offset indices inside it claim.

## The attack

1. Build a tx with two `Ed25519Program` instructions.
2. Instruction A (position -1 relative to the transfer call): raw
   bytes crafted by hand — `pubkey = config.signer_key` (public,
   sitting in the Config account), `message` = whatever
   `TransferParams` you want. Its offset header's three
   `*_instruction_index` fields point at instruction B, not `0xFFFF`.
3. Instruction B: a completely real, valid ed25519 instruction —
   signed by a throwaway key over throwaway data. Costs nothing to
   produce, has nothing to do with Cashmere.
4. Runtime verifies B (passes — it's a real signature). Program reads
   A's raw bytes directly, compares against `config.signer_key` /
   expected `TransferParams` — matches, because the attacker wrote
   those exact bytes into A themselves. No signature ever covered them.

Result: `transfer`/`transfer_v2` callable with arbitrary
`destination_domain`/`fee`/`deadline`/`fee_is_native`, no real backend
signature required, backend key never touched.

## Actual impact (being honest about it)

Every value transfer in `pre_transfer` goes caller -> fee_collector /
gas_drop_collector. Nothing already sitting in Cashmere's accounts can
be drained through this path.

What it does allow:
- `fee = 0` on every transfer, unlimited — costs Cashmere nothing
  *today* since `fee_bp` is already 0, but removes the control
  entirely, not just weakens it.
- `gas_drop_amount` up to the admin-configured cap (not zero) — real
  SOL/USDC out of the gas_drop_collector accounts, protocol's expense.
- `deadline` arbitrarily far out — the quote never expires.

Severity: critical, because the control is gone, not degraded. The
day `fee_bp` moves off zero, or Solana volume grows, or anything else
leans on this same `verify_ed25519_ix` for authorization, this stops
being theoretical.

## Second, smaller finding

Signed payload shape is identical across EVM/Solana/Aptos/Sui: local
domain, destination domain, fee, deadline, native/version flag. None
of the four commit to amount, recipient, or a signature-specific
nonce. Sui code has a comment acknowledging `gas_on_destination is not
signed` — may well be intentional. As-is: one signed fee quote is
valid for any amount/recipient/gas-drop, replayable until deadline, by
anyone who's seen it in a past transaction. Worth a second look
regardless, and more so on Solana where the first bug also makes the
deadline itself attacker-controlled.

## Fix

```rust
require!(
    ed25519_offsets.signature_instruction_index == u16::MAX
    && ed25519_offsets.public_key_instruction_index == u16::MAX
    && ed25519_offsets.message_instruction_index == u16::MAX,
    SignatureVerificationError::InvalidSignatureData
);
```

Standard Solana security guidance for this exact verification pattern.

## To close the proof fully

Haven't seen `pre_transfer.rs` itself (wherever it actually lives —
`instructions/transfer/common.rs` based on the `use super::pre_transfer`
import in `transfer_ix.rs`/`transfer_v2_ix.rs`). Would confirm
`pub_key = config.signer_key` and `msg = serialized TransferParams` are
exactly what's passed in, matching the EVM/Aptos/Sui pattern. Ask for
it if/when they respond.
