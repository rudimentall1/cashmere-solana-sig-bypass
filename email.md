Hey,

I've been going through the CashmereCCTP contracts in your public repo (cashmere-prod/contracts-prod) over the past few days, across all four chains you support, and found something in the Solana program I think you'll want to look at soon. Also a smaller thing that shows up consistently across all four implementations, which might just be intentional on your end — flagging it anyway.

Couldn't find a bug bounty program or a security.txt anywhere, so I'm just emailing directly. Nothing here was tested against mainnet, no funds touched, no live transactions — everything below I confirmed by reading the code and it's all reproducible on devnet if you want to check it yourselves before doing anything else.

The main one: your Solana signature verification is bypassable.

It's in utils/ed25519.rs, the verify_ed25519_ix function, called from pre_transfer in instructions/transfer/common.rs, which is what both transfer_ix and transfer_v2_ix rely on for checking that a transfer was actually authorized by your backend signer.

The way Solana signature verification works, you can't check an ed25519 signature inside your own program directly — you rely on a separate native instruction (Ed25519Program) sitting somewhere in the same transaction, which does the actual crypto check, and then your program just has to confirm that instruction is really there and really says what you expect. That confirmation step is where the bug is. Your code reads the instruction sitting right before yours (position -1) and pulls out a few offset fields from it — where the pubkey is, where the signature is, where the message is, and which "instruction index" each of those point to. It checks that those three instruction-index values match each other. What it doesn't check is that they equal 0xFFFF, which is the special value meaning "these offsets point into this same instruction." Without that check, someone can point those indices at a completely different instruction elsewhere in the transaction.

So here's the actual attack: you build a transaction with two ed25519 instructions. The first one, sitting at position -1 relative to your transfer call, has its offset header pointing at the second one. The second one is a totally real, valid signature — just signed by the attacker's own throwaway key over whatever garbage message they want, costs them nothing to produce. Solana's runtime checks that second instruction and it passes, because it is a real signature, just not one that has anything to do with Cashmere. Then your program comes along, looks at instruction #1 (not #2), and just reads the raw bytes sitting there — which the attacker wrote themselves, unsigned, straight into the pubkey/message fields. It compares those bytes against config.signer_key and the serialized TransferParams it's expecting, and of course they match, because the attacker put the exact bytes there on purpose. No actual signature ever covered them.

End result: someone can call transfer or transfer_v2 with whatever destination_domain / fee / deadline / fee_is_native they want, without ever having a real signature from your backend, and without your private key ever being compromised. It's a full bypass of that whole check, not a weakness in it.

Now, to be straight with you about what this actually gets someone — I don't want to overstate it. I went through pre_transfer line by line and every transfer of value in there goes from the caller to your fee_collector / gas_drop_collector accounts, never the other way. So nobody's draining anything that's already sitting in your accounts through this. What it does let someone do is set fee to 0 and skip your relayer fee entirely on however many transfers they want. And I noticed in your docs the fee is already set to 0 bp right now anyway, so as of today this doesn't cost you anything real. I'm still calling it critical because the actual security control is just gone, not degraded — the moment you turn fee_bp up, or Solana volume grows, or you build anything else that leans on this same verify_ed25519_ix function for authorization, this becomes a real problem instead of a theoretical one.

If you want to check it yourselves: spin up the program on devnet, run initialize + set_signer_key with a test key, then build a transaction the way I described above — one crafted ed25519 instruction with offsets pointing at a second real one — and call transfer_ix. If it goes through without ever using your actual signer key, that confirms it.

Fix is simple, just add a check that all three instruction-index fields equal u16::MAX before trusting anything else in that instruction:

require!(
 ed25519_offsets.signature_instruction_index == u16::MAX
 && ed25519_offsets.public_key_instruction_index == u16::MAX
 && ed25519_offsets.message_instruction_index == u16::MAX,
 SignatureVerificationError::InvalidSignatureData
);

Second thing, smaller, medium severity at most: I compared what actually gets signed across all four chains — EVM, Solana, Aptos, and Sui — and it's the same shape everywhere: local domain, destination domain, fee, deadline, and a native/version flag. None of them include the amount, the recipient, or any kind of nonce tied to that specific signature. Your Sui code even has a comment saying as much ("gas_on_destination is not signed"), so this might just be how you designed it on purpose. But as it stands, one signed fee quote from your backend is valid for any amount, any recipient, any gas drop amount, and can be reused as many times as someone wants within the deadline window, by anyone who happens to see it in a past transaction's calldata. Worth a second look, especially since on Solana the deadline itself becomes attacker-controlled too once the first bug is in play.

That's everything. Not looking for a payout here — I know your Solana side isn't carrying much value right now and I don't think you have a formal bounty program set up, so I'm not expecting one. Would just appreciate knowing it landed somewhere and got looked at, and a mention if you ever write anything public about it. Let me know if you want more detail on either one.

[name]
x.com/overline2024
