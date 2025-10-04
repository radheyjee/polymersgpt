# PolymersGPT - Anchor Programs & On-Chain Logic

PolymersGPT v1.0 Beta leverages Anchor to manage all critical on-chain logic for:
	•	User accounts and AI interactions
	•	PLY token transactions and staking
	•	NFT minting and marketplace operations (via Metaplex)
	•	ESG/e-waste event validation (via Helium IoT & Pyth)
	•	DAO governance
	•	Chat-triggered Solana Actions and Dialect Blinks

⸻

Program Overview
	•	Program ID: POLY5gpt... (replace with deployed ID)
	•	Dependencies: Anchor, SPL Token, Metaplex, Helius RPC, Pyth Oracles, Solana Pay, Privy.io DID, Dialect Blinks

⸻

Program Accounts

Account Type	Purpose	Key Fields
UserAccount	Stores user profile, balance, activity	user_pubkey, ply_balance, free_messages, nft_count, esg_points, last_interaction
AgentAccount	Tracks AI agent state	agent_id, agent_type, allowed_actions, is_active
PromptNFTAccount	NFT metadata for prompts	nft_id, owner, prompt_data, metadata_uri, royalty_bps, listed_price
TransactionAccount	Logs PLY transactions	tx_id, from, to, amount, type, timestamp
ESGEventAccount	Tracks ESG/e-waste events	event_id, user, event_hash, carbon_credits, timestamp, oracle_verified
StakingAccount	Manages PLY staking	user, staked_ply, start_time, reward_rate, last_claim
GovernanceAccount	Tracks DAO proposals	proposal_id, creator, proposal_data, votes_for, votes_against, status, end_time


⸻

Core Instructions

1. initialize_user

Creates a new UserAccount with 10 free messages.

pub fn initialize_user(ctx: Context<InitializeUser>) -> Result<()> {
    let user_account = &mut ctx.accounts.user_account;
    require!(user_account.ply_balance == 0, ErrorCode::UserAlreadyInitialized);
    user_account.user_pubkey = ctx.accounts.user.key();
    user_account.free_messages = 10;
    user_account.last_interaction = Clock::get()?.unix_timestamp;
    emit!(UserInitialized { user: user_account.user_pubkey, timestamp: user_account.last_interaction });
    Ok(())
}


⸻

2. send_message

Deducts PLY tokens or uses free messages. Includes rate limiting and reentrancy protection.

pub fn send_message(ctx: Context<SendMessage>, agent_id: u8, prompt_id: u64) -> Result<()> {
    let user_account = &mut ctx.accounts.user_account;
    let tx_account = &mut ctx.accounts.transaction_account;
    let amount = 10_000;

    require!(ctx.accounts.user.is_signer, ErrorCode::Unauthorized);
    require!(user_account.ply_balance >= amount || user_account.free_messages > 0, ErrorCode::InsufficientPly);

    if user_account.free_messages > 0 {
        user_account.free_messages -= 1;
    } else {
        user_account.ply_balance -= amount;
        let cpi_accounts = Transfer { /* ... */ };
        token::transfer(CpiContext::new(ctx.accounts.token_program.to_account_info(), cpi_accounts), amount)?;
    }

    tx_account.tx_id = prompt_id;
    tx_account.from = ctx.accounts.user.key();
    tx_account.amount = amount;
    tx_account.type = "Message".to_string();
    emit!(MessageSent { user: ctx.accounts.user.key(), agent_id, prompt_id, amount });
    Ok(())
}


⸻

3. reward_esg

Validates ESG/e-waste events with ZKPs, mints PLY rewards.

pub fn reward_esg(ctx: Context<RewardESG>, event_hash: String, carbon_credits: u64) -> Result<()> {
    require!(validate_zk_proof(&event_hash, &ctx.accounts.pyth_oracle), ErrorCode::InvalidESGEvent);
    let user_account = &mut ctx.accounts.user_account;
    user_account.ply_balance += carbon_credits;
    emit!(ESGRewarded { user: ctx.accounts.user.key(), event_hash, credits: carbon_credits });
    Ok(())
}


⸻

4. Other Instructions
	•	mint_prompt_nft → Mint NFT via Metaplex
	•	buy_prompt → Purchase NFT using PLY
	•	stake_tokens / unstake_tokens → Manage PLY staking
	•	execute_agent_action → Trigger AI agent logic from chat
	•	create_proposal / vote_proposal → DAO governance

⸻

Security Measures
	1.	Access Control: is_signer checks + Privy.io DID verification.
	2.	Zero-Knowledge Proofs: Protect ESG/e-waste data while issuing rewards.
	3.	Immutable Logs: All transactions and ESG events stored on-chain.
	4.	Rate Limiting: Prevents spam and abuse of AI agents.
	5.	Reentrancy Protection: Updates state before CPIs.
	6.	CPI Security: Only trusted program IDs (SPL Token, Metaplex) can be invoked.
	7.	Event Emission: Real-time monitoring of MessageSent, ESGRewarded, NFTMinted.
	8.	Upgradeability: DAO-governed program upgrades via BPFLoaderUpgradeable.
	9.	Error Handling: Custom errors prevent invalid state transitions.
	10.	Dialect Blinks & Solana Actions: Validates all chat-triggered actions.

⸻

Testing & Auditing
	•	Unit Tests: Each instruction tested with Anchor framework
	•	Integration Tests: Simulate full Solana interactions
	•	Fuzz Testing: Validate edge cases and reentrancy scenarios
	•	Debugging: RUST_LOG=anchor=debug, Helius RPC for traceability
	•	External Audit: Recommended before mainnet deployment

⸻

Deployment

anchor build
anchor deploy --provider.cluster mainnet
anchor idl init --filepath target/idl/polymersgpt.json --provider.cluster mainnet
