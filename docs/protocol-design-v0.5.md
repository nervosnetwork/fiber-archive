Fiber Network Protocol Design V0.5
=================

* [Fiber Network Protocol Design V0.5](#fiber-network-protocol-design-v05)
    * [Open Channel](#open-channel)
        * [Off-chain Negotiate](#off-chain-negotiate)
        * [On-chain Scripts](#on-chain-scripts)
    * [Off-chain Payment](#off-chain-payment)
        * [Straight Payment](#straight-payment)
        * [HTLC Payment](#htlc-payment)
            * [Update commitment to a new version](#update-commitment-to-a-new-version)
            * [Settle HTLC](#settle-htlc)
            * [HTLC Proof](#htlc-proof)
    * [Deposit](#deposit)
        * [Create Deposit Proof Cell](#create-deposit-proof-cell)
        * [Recycle Deposit Proof Cell](#recycle-deposit-proof-cell)
        * [Limit the settled_access data size](#limit-the-settled_access-data-size)
    * [Withdraw](#withdraw)
        * [Create Withdrawal Proof Cell](#create-withdrawal-proof-cell)
        * [Recycle Withdrawal Proof Cell](#recycle-withdrawal-proof-cell)
        * [Limit the settled_access data size](#limit-the-settled_access-data-size-1)
    * [Bilaterally Close Channel](#bilaterally-close-channel)
    * [Unilaterally Close Channel](#unilaterally-close-channel)
        * [Close Channel](#close-channel)
        * [Update Channel](#update-channel)
            * [Update Version](#update-version)
            * [Update HTLC](#update-htlc)
            * [Update Access](#update-access)
        * [Settle Channel](#settle-channel)
    * [Summarize Channel State Transition](#summarize-channel-state-transition)
    
# Fiber Network Protocol Design V0.5

> Refer to [design v0.4](https://github.com/nervosnetwork/fiber/blob/master/docs/zh-cn/architecture-design-v0.4.md)

> For information about the upgraded design, refer to: https://github.com/nervosnetwork/fiber/issues/3

This article describes the off-chain interactions and on-chain transactions of fiber payment channel. To prevent the article from being too long, some context may be missing. In order to describe the whole protocol clearly, more specifics need be added in the future. 

## Open Channel

Alice and Bob agreed to open a channel for mutual payment in the next period of time.

### Off-chain Negotiate

1. Alice sended a open-channel-request to Bob

   ```
   struct OpenChannelRequest {
       requester_pubkey_hash: Byte32;
       requester_deposit_amount: uint128;
       asset_type: AssetType;
       since: uint16;  // the input since of channel cell when transit state to SETTLING from CLOSING
       max_update_times: uint8;  // since and max_update_times determines the max length of closing period
   }
   
   enum AssetType {
       CKB(byte),
       SUDT(Byte32)
   }
   ```

2. Bob agreed and responsed to Alice

   ```
   struct OpenChannelResponse {
       responser_pubkey_hash: Byte32;
       responser_deposit_amount: uint128;
   }
   ```

3. Alice constructed a open-channel-tx, signed it, sended raw-tx and signature to Bob

   ```
   - inputs
     - alice provide capacity cell
       - capacity: > 100 CKB
       - lockscript: alice
     - bob provide capacity cell
       - capacity: > 200 CKB
       - lockscript: bob
   - outputs
     - channel cell
       - typescript: PCT
         - args: channel_id // hash(inputs[0], outputs_index)
       - lockscript: anyone can unlock
       - data
         - fixed
           - asset_type: CKB
           - since: 12000 
           - pubkey_hashes: [pubkey_hash of alice, pubkey_hash of bob]
         - dynamic
           - status: OPENING
           - version: 0
           - balances: (100 CKB, 200 CKB)
           - htlcs: none
           - settled_htlcs: none
           - settled_access: none
           - remain_update_times: (10, 10)
     - asset cell
       - capacity: 300 CKB
       - lockscript: PCAL
         - args: hash of PCT script
     - ckb change cell
       - lockscript: alice
     - ckb change cell
       - lockscript: bob
       
   struct PaymentChannelCellData {
        status: ChannelStatus;
        version: uint64;
        pubkey_hashes: (Byte32, Byte32);
        asset_type: AssetType;
        balances: (uint128, uint128);
        htlcs: Option<Htlcs>;
        settled_htlcs: Option<Bitmap>;
        settled_access: Option<SettledAccess>;
        remain_update_times: (uint8, uint8);
        since: uint16;  // the input since of channel cell when transit state to SETTLING from CLOSING 
   }
   
   enum ChannelStatus {
        OPENING,
        CLOSING,
        SETTLING
   }
   
   enum AssetType {
       CKB(byte),
       SUDT(Byte32)
   }
   
   struct Htlcs {
        length: uint16;
        merkle_root: Byte32;
   }
   
   struct SettledAccess {
        deposits: [Outpoints];
        withdrawals: [Outpoints];
   }

4. Bob verified the received tx, signed and sended it.

### On-chain Scripts

1. PCT (Payment Channel Cell Typescript): manage channel cell

    - verify channel_id ==  hash(inputs[0], outputs_index)
   > Other states should be verified in off-chain negotiate stage

2. PCAL (Payment Channel Asset Cell Lockscript): manage channel asset cell

   - only unlocked if inputs contained related channel cell 
   > Lockscript in tx outputs will not be executed

## Off-chain Payment

### Straight Payment

Alice wanted to pay 10 CKB to Bob:

1. Alice constructed a new payment commitment, signed and sended it to Bob

   ```
   struct PaymentCommitmentToSign {
        channel_id: Byte32;
        version: uint64;
        balances: (uint128, uint128);  // (participant0_balance, participant0_balance);
        htlcs: Option<Htlcs>;
        settled_accesses: Option<SettledAccess>;
   }
   
   // Htlcs records the HTLC payments in commitment
   struct Htlcs {
        length: uint16;  // length of htlcs
        merkle_root: Byte32;  // merkle tree: https://github.com/nervosnetwork/merkle-tree
   }
   
   // SettledAccess records the settled deposits/withdrawals in commitment
   struct SettledAccess {
        deposits: [Outpoints];
        withdrawals: [Outpoints];
   }
   
   struct PaymentCommitment {
        raw_data: PaymentCommitmentToSign;
        signatures: (Byte32, Byte32); // (participant0_signature, participant1_signature)
   }
   
   // commitment alice sended to bob
   {
        raw_data: {
             channel_id: 0x9292a49e5cfd3653b242ec93a39a1d9b183ab0e4efd4fc51045b42f10e93cb03,
             version: 1,
             balances: [90 CKB, 210 CKB],
             htlcs: none,
             settled_access: none,
        },
        signatures: [alice_signature, none]
   }
   ```

2. Bob verified the commitment, signed it and sended signature to Alice

   Bob updated the latest commitment version

3. Alice received the signature, verified and updated the latest commitment version

   If Bob didn't send signature back, Alice would not sign any new commitment. Alice also could unilaterally close channel.

### HTLC Payment

Bob wanted to pay 10 CKB with HTLC to Alice:

#### Update commitment to a new version

1. Bob constructed a new payment commitment, signed and sended it to Bob
   - Merkle tree for htlcs: https://github.com/nervosnetwork/merkle-tree
```
// commitment bob sended to alice
{
     raw_data: {
          channel_id: 0x9292a49e5cfd3653b242ec93a39a1d9b183ab0e4efd4fc51045b42f10e93cb03,
          version: 2,
          balances: [90 CKB, 200 CKB],
          htlcs: {
               length: 1,
               // https://github.com/nervosnetwork/merkle-tree
               merkle_root: 0x8292a49e5cfd3653b242ec93a39a1d9b183ab0e4efd4fc51045b42f10e93cb03,
          },
          settled_access: none,
     },
     signatures: (none, bob_signature)
}

// Htlc raw data should be included in commitment/htlcs/merkle_root
struct Htlc {
     amount: uint128;
     to: uint8;
     hash_lock: Byte32;
     last_unlock_block_number: uint64;
}
```

2. Alice signed the commitment and sended signature to Bob

   Alice updated the latest commitment version.

3. Bob received the  signature and updated the latest commitment version

   If Alice didn't send signature back, Bob would not sign any new commitment. Bob also could unilaterally close channel.

#### Settle HTLC

If Alice revealed the preimage to Bob:

1. Alice constructed a new payment commitment, signed and sended it to Bob
   
   ```
   // HTLC payment should be settled from htlcs to balances
   {
     raw_data: {
          channel_id: 0x9292a49e5cfd3653b242ec93a39a1d9b183ab0e4efd4fc51045b42f10e93cb03,
          version: 3,
          balances: [100 CKB, 200 CKB],
          htlcs: none,
          settled_access: none,
     },
     signatures: (alice_signature, none)
   }
   ```

2. Bob signed the new commitment, sended signature to Alice, and updated the latest commitment version

3. Alice verified the received signature and updated the latest commitment version

   If Bob did not sign and send signature to Alice, Alice could unilaterally close channel and settle the HTLC on-chain.



If Alice didn't reveal the preimage to Bob, and the HTCL was timeout

1. Bob constructed a new payment commitment, signed and sended it to Alice

   ```
   // HTLC payment should refund
   {
        raw_data: {
             channel_id: 0x9292a49e5cfd3653b242ec93a39a1d9b183ab0e4efd4fc51045b42f10e93cb03,
             version: 3,
             balances: [90 CKB, 210 CKB],
             htlcs: none,
             settled_access: none,
        },
        signatures: (none, bob_signature)
   }
   ```

2. Alice signed the new commitment, sended signature to Bob, and updated the latest commitment version

3. Bob verified the received signature and updated the latest commitment version

   If Alice didn't send signature back, Bob would not sign any new commitment. Bob also could unilaterally close channel.

#### HTLC Proof

If participant revealed the preimage, but the counterparty refused to sign the new payment commitment. The participant could send a create-htlc-proof-tx on-chain, and the htlc could be settled on-chain when closing channel.

```
// create htlc proof tx
- inputs
  - provide capacity cell
- outputs
  - htlc proof cell
    - data
      - preimage
    - lockscript
      - args: preimage collector id
      - logic
        - can not be destroyed unless merged into preimage collector cell
```

We designed preimage collector cell to recycle htlc proof cell, a recycle-htlc-proof-tx could be constructed as follow:
  - Sparse Merkle tree for preimages: https://github.com/jjyr/restricted-sparse-merkle-tree
```
// recycle htlc proof tx
- header_deps
  - header deps for htlc proof cell
- inputs
  - htlc proof cell
    - data
      - preimage
    - lockscript
      - args: preimage cell id
      - logic
        - can not be destroyed unless merged into preimage cell	
  - preimage collector cell
    - data
      - merkle_root  // https://github.com/jjyr/restricted-sparse-merkle-tree
        - key: lock_hash
        - value: (unlock_preimage, unlock_time)
    - typescrit
      - args: collector_id // similar to type_id
      - logic
        - verify htlc proof merged into merkle tree
        - input since should be greater than 5 when consumed
- outputs
  - preimage collector cell
```

Both htlc proof cell and preimage collector cell can be used to settle htlc when closing channel.

## Deposit

Alice wanted to deposit 100 CKB to channel:

### Create Deposit Proof Cell

1. Alice sended a deposit-channel-tx on-chain

   ```
   // deposit channel tx
   - inputs
     - provide capacity cell
       - capacity: > 100 CKB
       - lockscript: alice
     - channel cell
       - typescript: PCT
         - args: channel_id
         - data
           - status: OPENING
     - asset cell
       - capacity: 300 CKB
       - lockscript: PCAL
   - outputs
     - channel cell
       - typescript: PCT
         - args: channel_id
       - data
         - status: OPENING
     - asset cell
       - capacity: 400 CKB
       - lockscript: PCAL
     - deposit proof cell
       - typescript: DPT
         - args: channel_id
       - data
         - asset_type: CKB
         - amount: 100 CKB
         - depositor: 0  // index of pubkey_hashes
   
   ```
   
   #### on-chain scripts
   - PCT
     - verify deposit channel state change
       - status should be OPENING, and channel cell not changed
       - asset cell should be deposited
   - PCAL
     - verify inputs contained related channel cell
   - DPT
     - verify self script args == PCT args
     - verify cell data asset_type && amount
   
2. Alice constructed a new payment commitment, signed and sended it to Bob

   ```
   {
        raw_data: {
             channel_id: 0x9292a49e5cfd3653b242ec93a39a1d9b183ab0e4efd4fc51045b42f10e93cb03,
             version: 4,
             balances: [200 CKB, 200 CKB],
             htlcs: none,
             settled_access: {
                  deposits: [outpoint of deposit proof cell,],
                  withdrawals: [],
             },
        },
        signatures: (alice_signature, none)
   }
   ```

3. Bob verified and signed the received commitment, sended signature to Alice

   Bob updated the latest commitment version

4. Alice verified the received signature and updated the latest commitment version

   If Bob didn't send signature back, Alice would not sign any new commitment. Alice also could unilaterally close channel.

### Recycle Deposit Proof Cell

1. Alice destroyed deposit-proof-cell and recycled capacity, then constructed a new payment commitment, signed and sended it to Bob

   ```
   {
        raw_data: {
             channel_id: 0x9292a49e5cfd3653b242ec93a39a1d9b183ab0e4efd4fc51045b42f10e93cb03,
             version: 5,
             balances: [200 CKB, 200 CKB],
             htlcs: none,
             settled_access: none,
        },
        signatures: (alice_signature, none)
   }
   ```

2. Bob verified and signed the received commitment, sended signature to Alice

   Bob updated the latest commitment version

3. Alice verified the received signature and updated the latest commitment version

   If Bob didn't send signature back, Alice would not sign any new commitment. Alice also could unilaterally close channel.

### Limit the `settled_access` data size

In case of oversized commitment, participants should limit the max length of `deposits`. In other words, depositor should recycle deposit-proof-cell at the right time.

## Withdraw

Alice wanted to withdraw 100 CKB from channel:

### Create Withdrawal Proof Cell

1. Alice constructed and signed a withdrawal commitment, sended it to Bob

   ```
   struct WithdrawalCommitmentToSign  {
        channel_id: Byte32;
        amount: uint128;
        withdrawer: uint8; // index of pubkey_hashes
   }
   
   struct WithdrawalCommitment  {
        raw_data: WithdrawalCommitmentToSign;
        signatures: (Bytes, Bytes); // [participant0_signature, participant1_signature]
   }
   
   // commitment alice sended to bob
   {
        raw_data: {
             channel_id: 0x9292a49e5cfd3653b242ec93a39a1d9b183ab0e4efd4fc51045b42f10e93cb03,
             amount: 100,
             withdrawer: 0,
        },
        signatures: [alice_signature, none]
   }
   ```

2. Bob verified and signed the commitment, sended the signature to Alice

3. Alice sended a withdraw-channel-tx on-chain

   Alice should send this tx before a certain time, otherwise Bob should close channel.

   ```
   - inputs	
     - channel cell
       - typescript: PCT
         - args: channel_id
       - data
         - status: OPENING
     - asset cell
       - capacity: 400 CKB
       - lockscript: PCAL
   - outputs
     - channel cell
       - typescript: PCT
         - args: channel_id
       - data
         - status: OPENING
     - asset cell
       - capacity: 300 CKB
       - lockscript: PCAL
     - withdrawal proof cell
       - typescript: WPT
         - args: channel_id
       - data  
         - amount: 100 CKB
         - withdrawer: 0  // index of pubkey_hashes
     - recipient cell
       - capacity: 100 CKB
       - lockscript: alice
   - witnesses
     - withdrawal_commitment
   ```
   #### on-chain scripts
   - PCT
     - verify withdrawal commitment
     - verify asset cell withdrawal
   - PCAL
     - verify inputs contained related channel cell
   - WPT
     - verify self script args == PCT args
     - verify cell data amount && withdrawer 
     - this cell can not be consumed before N blocks

4. Bob constructed and signed a new payment commitment, sended it to Alice

   ```
   {
	    raw_data: {
           channel_id: 0x9292a49e5cfd3653b242ec93a39a1d9b183ab0e4efd4fc51045b42f10e93cb03,
           version: 6,
           balances: [100 CKB, 200 CKB],
           htlcs: none
           settled_access: {
                deposits: [],
                withdrawals: [outpoint of withdrawal proof cell,],
           }
	    },
	    signatures: (none, bob_signature)
   }
   ```

5. Alice received and verified the commitment, signed it and sended signature to Bob

   Alice updated the latest commitment version.

6. Bob verified the received signature and updated the latest commitment version

   If Alice didn't send signature back, Bob would unilaterally close channel in case of withdrawal proof cell was destroyed after N blocks.

### Recycle Withdrawal Proof Cell

1. Alice destroyed withdrawal-proof-cell and recycled capacity, then constructed a  new payment commitment, signed and sended it to Bob

   ```
   {
        raw_data: {
             channel_id: 0x9292a49e5cfd3653b242ec93a39a1d9b183ab0e4efd4fc51045b42f10e93cb03,
             version: 7,
             balances: [100 CKB, 200 CKB],
             htlcs: none,
             settled_access: none,
        },
        signatures: (alice_signature, none)
   }
   ```

2. Bob verified and signed the received commitment, sended signature to Alice

   Bob updated the latest commitment version

3. Alice verified the received signature and updated the latest commitment version

   If Bob didn't send signature back, Alice would not sign any new commitment. Alice also could unilaterally close channel.

### Limit the `settled_access` data size

In case of oversized commitment, participants should limit the max length of `withdrawals`. In other words, withdrawer should recycle withdrawal-proof-cell at the right time.

## Bilaterally Close Channel

Alice wanted to close channel:

1.  Alice constructed a bilaterally close commitment, signed and sended it to Bob

   ```
   struct BilaterallyCloseChannelCommitmentToSign {
        channel_id: Byte32;
        asset_type: AseetType;
        balances: (uint128, uint128);
   }
   
   struct BilaterallyCloseChannelCommitment {
        raw_data: BilaterallyCloseChannelCommitmentToSign;
        signatures: (Bytes, Bytes);
   }
   
   // commitment alice sended to bob
   {
        raw_data: {
             channel_id: 0x9292a49e5cfd3653b242ec93a39a1d9b183ab0e4efd4fc51045b42f10e93cb03,
             asset_type: CKB,
             balances: (100, 200)
        },
        signatures: (signature by alice, none)
   }
   ```

2. Bob received, verified, and signed the commitment, sended a  bilaterally-close-channel-tx on-chain

   ```
   - inputs
     - provide fee cell
     - channel cell
       - data
         - status: OPENING
       - typescript: PCT
     - asset cell
       - capacity: 300 CKB
       - lockscript: PCAL
   - outputs
     - alice asset cell
       - capacity: 100 CKB
       - lockscript: alice
     - bob asset cell
       - capacity: 200 CKB
       - lockscript: bob
     - capacity change cell
   - witnesses
     - bilaterally_close_commitment
   ```
   #### on-chain scripts
   - PCT
     - verify status == OPENING
     - verify bilaterally close commitment
     - verify distributed assets
   - PCAL
     - verify inputs contained related channel cell
   
## Unilaterally Close Channel

### Close Channel

Alice sended a unilaterally-close-channel tx meanwhile submitted payment commitment:

```
- inputs
  - channel cell
    - typescript: PCT
    - lockscript: anyone can unlock
    - data
      - fixed
        - asset_type: CKB
        - since: 12000
        - pubkey_hashes: [pubkey_hash of alice, pubkey_hash of bob]
      - dynamic
        - status: OPENING
        - version: 0
        - balances: (100 CKB, 200 CKB)
        - htlcs: none
        - settled_htlcs: none
        - settled_access: none
        - remain_update_times: (10, 10)
- outputs
  - channel cell
    - typescript: PCT
    - lockscript: anyone can unlock
    - data
      - fixed
        - asset_type: CKB
        - since: 12000
        - pubkey_hashes: [pubkey_hash of alice, pubkey_hash of bob]
      - dynamic
        - status: CLOSING
        - version: 9
        - balances: (100 CKB, 100 CKB)
        - htlcs
          - length: 5
          - merkle_root: 0xffff5cfd3653b242ec93a39a1d9b183ab0e4efd4fc51045b42f10e93cb03ffff
        - settled_htlcs: none
        - settled_access
          - deposits: [outpoint0]
          - withdrawals: [outpoint0]
        - remain_update_times: (9, 10)
- witnesses  
  - unilaterally_close_channel_witness	
    - payment_commitment
      - raw_data
        - channel_id: 0x9292a49e5cfd3653b242ec93a39a1d9b183ab0e4efd4fc51045b42f10e93cb03
        - version: 9
        - balances: (100, 100)
        - htlcs
          - length: 5
          - merkle_root: 0xffff5cfd3653b242ec93a39a1d9b183ab0e4efd4fc51045b42f10e93cb03ffff
        - settled_access
          - deposits: [outpoint0]
          - withdrawals: [outpoint0]
      - signatures
    - closer signature: alice_signature
    
struct UnilaterallyCloseChannelWitness {
    payment_commitment: PaymentCommitment;
    closer_signature: Bytes;
}
```
#### on-chain scripts
- PCT
  - verify unilaterally close channel witness
  - verify channel cell state change
   
The channel status became `CLOSING` and states in payment commitment were stored in output channel cell data.

### Update Channel

Participants could send a update-channel-tx in this stage. A update-channel-tx could update payment commitment version, htlc proofs, access proofs or all of them.

```
struct UpdateChannelWitness {
     version: Option<UpdateVersionWitness>;
     htlc_proof: Option<UpdateHtlcProofsWitness>;
     access_proof: Option<UpdateAccessProofsWitness>;
}

struct UpdateVersionWitness {
     payment_commitment: PaymentCommitment;
     updater_signature: Bytes;
}

struct UpdateHtlcProofsWitness {
     raw_data: [Htcl];
     merkle_proof: MerkleProof; // A merkle proof can prove multiple nodes
     preimage_proofs: [PreimageProof];
     updater_signature: Bytes;
}

enum PreimageProof {
     DirectProof(PreimageDirectProof),
     PreimageCollector(PreimageCollectorProof),
}

// HTLC proof cell
struct PreimageDirectProof {
     cell_deps_index: uint8;
     header_deps_index: uint8;
}

struct PreimageCollectorProof {
     cell_deps_index: uint8; // index of preimage collector cell in cell deps
     merkle_proof: MerkleProof; // https://github.com/jjyr/restricted-sparse-merkle-tree
     raw_data: RevealedPreimage;
}

struct RevealedPreimage {
     preimage: Bytes;
     revealed_time: uint64;
}

struct UpdateAccessProofsWitness {
     cell_deps_index: [uint8]; // index of deposit proof cell or withdrawal proof cell in cell deps
     updater_signature: Bytes;
}
```

#### Update Version

The participant could update channel with newer commitment if the counterparty submitted the outdated version when closing channel.

```
// bob updated newer commitment
- inputs
  - channel cell
    - typescript: PCT
    - lockscript: anyone can unlock
    - data
      - fixed
        - asset_type: CKB
        - since: 12000
        - pubkey_hashes: [pubkey_hash of alice, pubkey_hash of bob]
      - dynamic
        - status: CLOSING
        - version: 9
        - balances: (100 CKB, 100 CKB)
        - htlcs
          - length: 5
          - merkle_root: 0xffff5cfd3653b242ec93a39a1d9b183ab0e4efd4fc51045b42f10e93cb03ffff
        - settled_htlcs: none
        - settled_access
          - deposits: [outpoint0]
          - withdrawals: [outpoint0]
        - remain_update_times: (9,10)
- outputs
  - channel cell
    - typescript: PCT
    - lockscript: anyone can unlock
    - data
      - fixed
        - asset_type: CKB
        - since: 12000
        - pubkey_hashes: [pubkey_hash of alice, pubkey_hash of bob]
      - dynamic
        - status: CLOSING
        - version: 10
        - balances: (100 CKB, 150 CKB)
        - htlcs
          - length: 3
          - merkle_root: 0x11115cfd3653b242ec93a39a1d9b183ab0e4efd4fc51045b42f10e93cb03ffff
        - settled_htlcs: none
        - settled_access
          - deposits: [outpoint0]
          - withdrawals: [outpoint0]
        - remain_update_times: (9, 9)
- witnesses  
  - update_channel_witness
	- update_version_witness
      - payment_commitment
        - raw_data
          - channel_id: 0x9292a49e5cfd3653b242ec93a39a1d9b183ab0e4efd4fc51045b42f10e93cb03
          - version: 10
          - balances: (100, 150)
          - htlcs
            - length: 3
            - merkle_root: 0x11115cfd3653b242ec93a39a1d9b183ab0e4efd4fc51045b42f10e93cb03ffff
          - settled_access
            - deposits: [outpoint0]
            - withdrawals: [outpoint0]
        - signatures
      - updater_signature: bob_signature
    - update_htlc_proofs_witness: none
    - update_access_proofs_witness: none
```
#### on-chain scripts
- PCT
  - verify update channel witness
  - verify channel cell state change
    - newer commitment was applied to channel cell
   
`version`, `balances`, `htlcs`, `settled_access` and `settled_htlcs` would be reset according to the new commitment and related proofs.

#### Update HTLC

If the htlc proof cell not existed on chain, the participant should send a create-htlc-proof-tx on-chain right now.

If the htlc proof cell existed on chain, but the related header was immature. Participants could wait for the header matured, and after that send a update-channel-tx to update htlc proof.

When participants sended a update-channel-tx, the submitted htlc proofs would be settled immediately in this tx(htlc amount would be added to balances). The settled htlcs would be recorded in `settled_htlcs` in case of duplicated settling.

```
// alice updated htlc proof
- header_deps
  - htlc0_proof_cell related header
- cell_deps
  - htlc0_proof_cell
- inputs
  - channel cell
    - typescript: PCT
    - lockscript: anyone can unlock
    - data
      - fixed
        - asset_type: CKB
        - since: 12000
        - pubkey_hashes: [pubkey_hash of alice, pubkey_hash of bob]
      - dynamic
        - status: CLOSING
        - version: 10
        - balances: (100 CKB, 150 CKB)
        - htlcs
          - length: 3
          - merkle_root: 0x11115cfd3653b242ec93a39a1d9b183ab0e4efd4fc51045b42f10e93cb03ffff
        - settled_htlcs: none
        - settled_access
          - deposits: [outpoint0]
          - withdrawals: [outpoint0]
        - remain_update_times: (9, 9)
- outputs
  - channel cell
    - typescript: PCT
    - lockscript: anyone can unlock
    - data
      - fixed
        - asset_type: CKB
        - since: 12000
        - pubkey_hashes: [pubkey_hash of alice, pubkey_hash of bob]
      - dynamic
        - status: CLOSING
        - version: 10
        - balances: (110 CKB, 150 CKB)
        - htlcs
          - length: 3
          - merkle_root: 0x11115cfd3653b242ec93a39a1d9b183ab0e4efd4fc51045b42f10e93cb03ffff
        - settled_htlcs: 0b00000001
        - settled_access
          - deposits: [outpoint0]
          - withdrawals: [outpoint0]
        - remain_update_times: (8, 9)
- witnesses  
  - update_channel_witness
    - update_version_witness: none
    - update_htlc_proofs_witness
      - raw_data
        - amount: 10
        - to: 0
        - hash_lock: 0x66665cfd3653b242ec93a39a1d9b183ab0e4efd4fc51045b42f10e93cb03ffff
        - last_unlock_block_number: 10000
      - htlc0 merkle proof
      - preimage_proofs
        - direct_proof
          - cell_deps_index: [0]
          - header_deps_index: [0]
      - updater signature: alice_signature
    - update_access_proofs_witness: none
```
#### on-chain scripts
- PCT
   - verify update channel witness
   - verify channel cell state change
     - htlc proof was settled to Alice's balance
   
#### Update Access

Access proofs(deposit/withdrawal proofs) also should be submitted in this stage, and the access would be directly settled to `balances` ,  related proofs would be recorded in `settled_access` in case of duplicated settling.

```
// alice updated access proof
- cell_deps
  - deposit_proof_cell0 (alice deposit 10)
  - deposit_proof_cell1 (alice deposit 10)
- inputs
  - channel cell
    - typescript: PCT
    - lockscript: anyone can unlock
    - data
      - fixed
        - asset_type: CKB
        - since: 12000
        - pubkey_hashes: [pubkey_hash of alice, pubkey_hash of bob]
      - dynamic
        - status: CLOSING
        - version: 10
        - balances: (110 CKB, 150 CKB)
        - htlcs
          - length: 3
          - merkle_root: 0x11115cfd3653b242ec93a39a1d9b183ab0e4efd4fc51045b42f10e93cb03ffff
        - settled_htlcs: none
        - settled_access
          - deposits: [outpoint0]
          - withdrawals: [outpoint0]
        - remain_update_times: (8, 9)
- outputs
  - channel cell
    - typescript: PCT
    - lockscript: anyone can unlock
    - data
      - fixed
        - asset_type: CKB
        - since: 12000
        - pubkey_hashes: [pubkey_hash of alice, pubkey_hash of bob]
      - dynamic
        - status: CLOSING
        - version: 10
        - balances: (130 CKB, 150 CKB)
        - htlcs
          - length: 3
          - merkle_root: 0x11115cfd3653b242ec93a39a1d9b183ab0e4efd4fc51045b42f10e93cb03ffff
        - settled_htlcs: 0b00000001
        - settled_access
          - deposits: [outpoint0, outpoint1, outpoint2]
          - withdrawals: [outpoint0]
        - remain_update_times: (7, 9)
- witnesses  
  - update_channel_witness
    - update_version_witness: none
    - update_htlc_proofs_witness: none
    - update_access_proofs_witness
      - cell_deps_index: [0, 1]
      - updater_signature: alice_signature
```
#### on-chain scripts
- PCT
   - verify update channel witness
   - verify channel cell state change
     - deposit proofs were settled to Alice's balance


### Settle Channel

If we could put `since` into channel_cell_input_since, then channel status would become `SETTLING` from `CLOSING` , in other words, closing period was over. Closing period was determined by update channel times and `since` parameter.

`since` makes sure that participants have enough time to submit any witness(including htlc proof). E.g. `since` could be 12000(greater than 4 epoches).

If participants had unproved htlcs(need  refund), it's time to submit them in this stage. And the `settled_htlcs` would record the settled unproved htlcs, so if the lowest `htlcs_length` bits of  `settled_htlcs` became 0b1, we knew all the htlcs was settled, channel cell and asset cell could be destroyed.

Alice submitted unproved htlc and got all her asset back.

```
- inputs
  - channel cell
    - since: 12000
    - typescript: PCT
    - lockscript: anyone can unlock
    - data
      - fixed
        - asset_type: CKB
        - since: 12000
        - pubkey_hashes: [pubkey_hash of alice, pubkey_hash of bob]
      - dynamic
        - status: CLOSING
        - version: 10
        - balances: (130 CKB, 150 CKB)
        - htlcs
          - length: 3
          - merkle_root: 0x11115cfd3653b242ec93a39a1d9b183ab0e4efd4fc51045b42f10e93cb03ffff
        - settled_htlcs: 0b00000001
        - settled_access
          - deposits: [outpoint0,  outpoint1, outpoint2]
          - withdrawals: [outpoint0]
        - remain_update_times: (7, 9)
  - asset cell
  	- capacity: 320 CKB
  	- lockscript: PCAL
- outputs
  - channel cell
    - typescript: PCT
    - lockscript: anyone can unlock
    - data
      - fixed
        - asset_type: CKB
        - since: 12000
        - pubkey_hashes: [pubkey_hash of alice, pubkey_hash of bob]
      - dynamic
        - status: SETTLING
        - version: 10
        - balances: (0 CKB, 0 CKB)
        - htlcs
          - length: 3
          - merkle_root: 0x11115cfd3653b242ec93a39a1d9b183ab0e4efd4fc51045b42f10e93cb03ffff
        - settled_htlcs: 0b00000011
        - settled_access
          - deposits: [outpoint0, outpoint1, outpoint2]
          - withdrawals: [outpoint0]
        - remain_update_times: (7, 9)
  - asset cell
    - capacity: 320 - 130 - 10 - 150 = 30 CKB
    - lockscript: PCAL
  - alice asset cell
    - capacity: 130 + 10 = 140 CKB
    - lockscript: alice
  - bob asset cell
    - capacity: 150 CKB
    - lockscript: bob  	
- witnesses  
  - settle_channel_witness
    - unproved_htlcs
      - raw_data
    	- amount: 10
        - to: 0
        - hash_lock: 0x66665cfd3653b242ec93a39a1d9b183ab0e4efd4fc51045b42f10e93cb03ffff
        - last_unlock_block_number: 10000
      - merkle_proof
      
struct SettleChannelWitness {
   unproved_htlcs: Option<UnprovedHtlcs>;
} 

struct UnprovedHtlcs {
   raw_data: [Htlc];
   merkle_proof: MerkleProof; // A merkle proof can prove multiple nodes
}
```
#### on-chain scripts
- PCT
  - verify settle channel witness
  - verify channel cell state change
    - verify input channel cell `since`
    - verify status changed to `SETTLING` from `CLOSING`
  - verify asset cell settlement
    - `balances` was settled to Alice and Bob
    - unproved htlc refunded to Alice
  
- PCAL
  - verify inputs contained related channel cell
   
At last, Bob submitted the last unproved htlc and got all his asset back.

```
- inputs
  - channel cell
    - typescript: PCT
    - lockscript: anyone can unlock
    - data
      - fixed
        - asset_type: CKB
        - since: 12000
        - pubkey_hashes: [pubkey_hash of alice, pubkey_hash of bob]
      - dynamic
        - status: SETTLING
        - version: 10
        - balances: (0 CKB, 0 CKB)
        - htlcs
          - length: 3
          - merkle_root: 0x11115cfd3653b242ec93a39a1d9b183ab0e4efd4fc51045b42f10e93cb03ffff
        - settled_htlcs: 0b00000011
        - settled_access
          - deposits: [outpoint0,  outpoint1, outpoint2]
          - withdrawals: [outpoint0]
        - remain_update_times: (7, 9)
  - asset cell
    - capacity: 30 CKB
    - lockscript: PCAL
- outputs
  - bob asset cell
    - capacity: 30 CKB
    - lockscript: bob
- witnesses  
  - settle_channel_witness
    - unproved_htlcs
      - raw_data
    	- amount: 30
        - to: 1
        - hash_lock: 0x77775cfd3653b242ec93a39a1d9b183ab0e4efd4fc51045b42f10e93cb03ffff
        - last_unlock_block_number: 10000
      - merkle_proof
```
#### on-chain scripts
- PCT
   - verify settle channel witness
   - verify channel cell state change 
     - verify `settled_htlcs` and channel cell can be destroyed
   - verify asset cell settlement
     - unproved htlc refund to Bob 

- PCAL
   - verify inputs contained related channel cell

## Summarize Channel State Transition

The payment channel state transition and the trigger witness in summary:

```
enum ChannelStateTransitionWitness {
    BilaterallyClose(BilaterallyCloseChannelCommitment);  // OPENING -> DESTROYED
    UnilaterallyClose(UnilaterallyCloseChannelWitness);  // OPENING -> CLOSING
    UpdateChannel(UpdateChannelWitness);  // CLOSING -> CLOSING
    SettleChannel(SettleChannelWitness);  // CLOSING -> SETTLING, SETTLING -> DESTROYED
}
```
