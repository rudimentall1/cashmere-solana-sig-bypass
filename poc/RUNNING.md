# Running the PoC

```
npm i @solana/web3.js tweetnacl ts-node typescript @types/node
solana-test-validator --reset   # separate terminal
anchor build && anchor deploy
solana-keygen new -o backend-signer.json --no-bip39-passphrase
# then call initialize_ix + set_signer_key_ix with that key
npx ts-node exploit.ts
```

Checklist before running:
- [ ] `solana-test-validator --reset` running
- [ ] `anchor build && anchor deploy` succeeded
- [ ] `backend-signer.json` created via `solana-keygen new`
- [ ] `initialize_ix` + `set_signer_key_ix` called with that pubkey
- [ ] account list in `transferIx.keys` matches `#[derive(Accounts)]` in
      `transfer_ix.rs` — adjust if it errors with "missing account"
