# Cashmere CCTP — Solana ed25519 signature bypass

Private tracking repo for a vuln found while reading through
cashmere-prod/contracts-prod. Not published anywhere, not shared
outside direct contact with the team.

## Status

- [x] Confirmed via code read (utils/ed25519.rs -> pre_transfer)
- [x] PoC written, not yet run against a live devnet deploy
- [ ] Emailed to the team
- [ ] Response
- [ ] Fix confirmed on mainnet
- [ ] Decide on public writeup

## Files

- `email.md` — what actually goes out to them. No PoC attached.
- `technical-notes.md` — full mechanism, in my own words, working notes
- `poc/exploit.ts` — working exploit, Solana devnet only. Private,
  stays private until (if ever) they confirm a mainnet fix.
- `TIMELINE.md` — dates, nothing else

## The short version

`verify_ed25519_ix` checks that the three `*_instruction_index` fields
in a native Ed25519Program instruction all match each other, but never
checks they equal `u16::MAX`. That means they can point at a
*different* ed25519 instruction elsewhere in the same transaction — one
with a real, cheaply-producible signature that has nothing to do with
Cashmere. Solana's runtime verifies that other instruction (passes,
it's real). The program then reads pubkey/message straight out of the
*first* instruction's raw data, which the attacker wrote by hand,
unsigned. No real signature from the backend key is ever checked.

Net effect on `transfer`/`transfer_v2`: arbitrary `fee` (currently
costs them nothing since fee_bp is 0 anyway), arbitrary
`gas_drop_amount` up to the admin cap (this one's real money out of the
gas_drop_collector accounts), and a deadline that never expires. No
funds sitting in their accounts get drained beyond that gas drop path.

Second, smaller thing: the signed payload (all 4 chains) never commits
to amount, recipient, or a signature-specific nonce. A quote is valid
for any amount/recipient and replayable until the deadline. Probably
intentional given the Sui comment, flagging anyway.
