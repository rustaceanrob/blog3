---
title: "Bitcoin Transactions from 0 and 1"
date: "Jan 29 2024"
description: "Constructing transations from raw bytes of data."

---

## Context

Some programmers adhere to the don't repeat yourself principle. I ignored that rule in every part of this assignment. In this code I construct some Bitcoin transactions from different sets of byte arrays. Some were signatures, others were hashes, all of them made me feel like I was in hell. The result of my code was a transaction that recorded my name in the Chaincode Signet blockchain. My name is sadly recorded as **rOB** instead of ROB in ASCII. You can find it [here](https://mempool.btcfoss.bitherding.com/tx/6d2e62cc7db1a88178eb4eb777f1243a9bf67e19a0f30a9f4ba80dc91097ec6a#flow=&vout=0). This was the week 2 exercise for the **Chaincode: Start Your Career in FOSS** program.

```rust
#![allow(unused)]
extern crate balance;
extern crate bitcoincore_rpc;
use balance::{OutgoingTx, WalletState};
use bitcoincore_rpc::bitcoin::{locktime, sighash};
use bitcoincore_rpc::{Auth, Client, RpcApi};
use bs58;
use hex;
use hmac_sha512::HMAC;
use num_bigint::BigUint; use ripemd::digest::{FixedOutput, Update};
use ripemd::digest::crypto_common::KeyInit;
// for modulus math on large numbers
use ripemd::{Ripemd160, Ripemd160Core};
use secp256k1::hashes::sha256;
use secp256k1::{Secp256k1, SecretKey, PublicKey, Message};
use sha2::{Sha256, Digest};
use std::borrow::Borrow;
use std::error::Error;
use std::hash::Hash;
use std::io::{Read, Write};
use std::ops::Add;
use std::path::PathBuf;
use std::str;

#[derive(Debug)]
pub enum SpendError {
    MissingCodeCantRun,
    // Add more relevant error variants
}

pub struct Utxo {
    script_pubkey: Vec<u8>,
    amount: u32,
}

pub struct Outpoint {
    txid: [u8; 32],
    index: u32,
}

// Given 2 compressed public keys as byte arrays, construct
// a 2-of-2 multisig output script. No length byte prefix is necessary.
fn create_multisig_script(keys: Vec<Vec<u8>>) -> Vec<u8> {
    let pk1 = &keys[0];
    let pk2 = &keys[1];
    let mut script: Vec<u8> = vec![];
    script.push(0x52); // PUSH 2
    let pk_1_size = pk1.len() as u8;
    script.push(pk_1_size);
    script.extend(pk1.iter());
    let pk_2_size = pk2.len() as u8;
    script.push(pk_2_size);
    script.extend(pk2.iter());
    script.push(0x52); // PUSH 2
    script.push(0xae); // check multisig
    return script;
}

// Given an output script as a byte array, compute the p2wsh witness program
// This is a segwit version 0 pay-to-script-hash witness program.
// https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki#p2wsh
fn get_p2wsh_program(script: &[u8], version: Option<u32>) -> Vec<u8> {
    let script_hash = single_hash(script.to_vec());
    if let Some(version) = version {
        let mut p2wsh: Vec<u8> = vec![version.try_into().unwrap(), 0x20];
        p2wsh.extend(script_hash.iter());
        return p2wsh
    } else {
        let mut p2wsh = vec![0, 0x20];
        p2wsh.extend(script_hash.iter());
        return p2wsh
    }
}

// Given an outpoint, return a serialized transaction input spending it
// Use hard-coded defaults for sequence and scriptSig
fn input_from_utxo(txid: &[u8], index: u32) -> Vec<u8> {
    let mut input: Vec<u8> = vec![];
    let position = index.to_le_bytes();
    input.extend(txid.iter());
    input.extend(position.iter());
    input.push(0x00);
    let seq = 0xffffffffu32.to_be_bytes();
    input.extend(seq.iter());
    // println!("Transaciton Input: {:?}", hex::encode(input.clone()));
    return input
}

// Given an output script and value (in satoshis), return a serialized transaction output
fn output_from_options(script: &[u8], value: u32) -> Vec<u8> {
    let mut ser_output: Vec<u8> = vec![];
    let sats = value.to_le_bytes();
    let pad = 0u32.to_be_bytes();
    let l = script.len() as u8;
    ser_output.extend(sats.iter());
    ser_output.extend(pad.iter());
    ser_output.push(l);
    ser_output.extend(script.iter());
    // println!("Output: {:?}", hex::encode(ser_output.clone()));
    return ser_output
}

// Given a Utxo object, extract the public key hash from the output script
// and assemble the p2wpkh scriptcode as defined in BIP143
// <script length> OP_DUP OP_HASH160 <pubkey hash> OP_EQUALVERIFY OP_CHECKSIG
// https://github.com/bitcoin/bips/blob/master/bip-0143.mediawiki#specification
fn get_p2wpkh_scriptcode(utxo: Utxo) -> Vec<u8> {
    let mut script_code: Vec<u8> = vec![];
    let first_half: [u8; 4] = 0x1976a914u32.to_be_bytes();
    script_code.extend(first_half.iter());
    // println!("Script Code Prefix {:?}", hex::encode(first_half.clone()));
    let pkh = &utxo.script_pubkey[2..];
    // println!("Public Key Hash in Script Code: {:?}", hex::encode(pkh.clone()));
    script_code.extend(pkh.iter());
    script_code.push(0x88);
    script_code.push(0xac);
    return script_code
}

// Compute the commitment hash for a single input and return bytes to sign.
// This implements the BIP 143 transaction digest algorithm
// https://github.com/bitcoin/bips/blob/master/bip-0143.mediawiki#specification
// We assume only a single input and two outputs,
// as well as constant default values for sequence and locktime
fn get_commitment_hash(
    outpoint: Outpoint,
    scriptcode: &[u8],
    value: u32,
    outputs: Vec<Utxo>,
) -> Vec<u8> {
    let mut commitment: Vec<u8> = vec![];
    // println!("\n==========================\n");
    // Version
    let version = 2u32.to_le_bytes();
    commitment.extend(version.iter()); //1
    // println!("Version: {:?}", hex::encode(version));

    // All TX input outpoints (only one in our case)
    let mut input_preimage = vec![]; 
    input_preimage.extend(outpoint.txid); 
    input_preimage.extend(outpoint.index.to_le_bytes());
    let hash_prevouts = double_hash(input_preimage.clone());
    commitment.extend(hash_prevouts.iter()); //2
    // println!("Hash Prevouts: {:?}", hex::encode(hash_prevouts));

    // All TX input sequences (only one for us, always default value)
    let ser_input_seq = double_hash(0xffffffffu32.to_be_bytes().to_vec());
    commitment.extend(ser_input_seq.iter()); //3
    // println!("Hash Sequence: {:?}", hex::encode(ser_input_seq));

    // Single outpoint being spent
    commitment.extend(input_preimage.iter()); // 4
    // println!("Outpoint: {:?}", hex::encode(input_preimage));

    // Scriptcode (the scriptPubKey in/implied by the output being spent, see BIP 143)
    commitment.extend(scriptcode.iter()); //5
    // println!("Script code: {:?}", hex::encode(scriptcode));

    // Value of output being spent
    let amount = value.to_le_bytes();
    let pad = 0u32.to_be_bytes();
    commitment.extend(amount.iter()); //6
    commitment.extend(pad.iter()); 
    // print!("Value: {:?}", hex::encode(amount));
    // println!("{:?}",  hex::encode(pad));
    

    // Sequence of output being spent (always default for us)
    let n_seq = 0xffffffffu32.to_be_bytes();
    commitment.extend(n_seq.iter()); //7
    // println!("nSequence: {:?}", hex::encode(n_seq));

    // All TX outputs
    let mut output_preimage: Vec<u8> = vec![];
    for output in outputs {
        let amount = output.amount.to_le_bytes();
        let pad = 0u32.to_be_bytes();
        output_preimage.extend(amount.iter());
        output_preimage.extend(pad.iter());
        let output_script_len = output.script_pubkey.len() as u8;
        output_preimage.push(output_script_len);
        output_preimage.extend(output.script_pubkey.iter());
    }
    // println!("Output Preimage (???): {:?}", hex::encode(output_preimage.clone()));
    let output_hash = double_hash(output_preimage);
    commitment.extend(output_hash.iter()); //8

    // Locktime (always default for us)
    let locktime = 0x00000000u32.to_be_bytes();
    commitment.extend(locktime.iter()); //9
    // println!("Locktime: {:?}", hex::encode(locktime));

    // SIGHASH_ALL (always default for us)
    let sighash = 1u32.to_le_bytes();
    commitment.extend(sighash.iter()); //10
    // println!("Sighash: {:?}", hex::encode(sighash));

    // println!("\nCommitment: {:?}", hex::encode(commitment.clone()));

    let d256 = double_hash(commitment);
    // println!("\nHash256: {:}\n", hex::encode(d256.clone()));

    return d256
}

// Given a JSON utxo object and a list of all of our wallet's witness programs,
// return the index of the derived key that can spend the coin.
// This index should match the corresponding private key in our wallet's list.
fn get_key_index(utxo: Utxo, programs: Vec<&str>) -> u32 {
    for (index, program) in programs.iter().enumerate() {
        if hex::decode(program).expect("Could not decode program").eq(&utxo.script_pubkey) {
            return index as u32
        }
    }
    return 0
}

// Given a private key and message digest as bytes, compute the ECDSA signature.
// Bitcoin signatures:
// - Must be strict-DER encoded
// - Must have the SIGHASH_ALL byte (0x01) appended
// - Must have a low s value as defined by BIP 62:
//   https://github.com/bitcoin/bips/blob/master/bip-0062.mediawiki#user-content-Low_S_values_in_signatures
fn sign(privkey: &[u8; 32], msg: Vec<u8>) -> Vec<u8> {
    // Keep signing until we produce a signature with "low s value"
    // We will have to decode the DER-encoded signature and extract the s value to check it
    // Format: 0x30 [total-length] 0x02 [R-length] [R] 0x02 [S-length] [S] [sighash]
    let secp = Secp256k1::new();
    let k = SecretKey::from_slice(privkey).expect("Could not conform to SecretKey");
    let message = Message::from_digest_slice(&msg).expect("Could not produce Message");
    let mut sig = secp.sign_ecdsa(&message, &k);
    sig.normalize_s();
    let der = sig.serialize_der();
    let mut strict_der_sig: Vec<u8> = vec![];
    strict_der_sig.extend(der.to_vec().iter());
    strict_der_sig.push(0x01);
    return strict_der_sig
}

// Given a private key and transaction commitment hash to sign,
// compute the signature and assemble the serialized p2pkh witness
// as defined in BIP 141 (2 stack items: signature, compressed public key)
// https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki#specification
fn get_p2wpkh_witness(privkey: &[u8; 32], msg: Vec<u8>) -> Vec<u8> {
    let mut witness: Vec<u8> = vec![];
    let sig = sign(privkey, msg);
    let sig_size = sig.len() as u8;
    // println!("Signature Size: {:?}", hex::encode([sig_size]));
    witness.push(sig_size);
    // println!("Signature: {:?}", hex::encode(sig.clone()));
    witness.extend(sig.iter());
    let pk = balance::derive_public_key_from_private(privkey);
    let pub_key_size = pk.len() as u8;
    // println!("Public Key Size: {:?}", hex::encode([pub_key_size]));
    witness.push(pub_key_size);
    // println!("Witness Public Key: {:?}", hex::encode(pk.clone()));
    witness.extend(pk.iter());
    return witness
}

// Given two private keys and a transaction commitment hash to sign,
// compute both signatures and assemble the serialized p2pkh witness
// as defined in BIP 141
// Remember to add a 0x00 byte as the first witness element for CHECKMULTISIG bug
// https://github.com/bitcoin/bips/blob/master/bip-0147.mediawiki
fn get_p2wsh_witness(privs: Vec<&[u8; 32]>, msg: Vec<u8>) -> Vec<u8> {
    //probably wrong
    let mut witness: Vec<u8> = vec![];
    // witness.push(0x01);
    witness.push(0x00);

    let sig_1 = sign(privs[0], msg.clone());
    let sig_1_len = sig_1.len() as u8;
    witness.push(sig_1_len);
    witness.extend(sig_1.iter());
    let pk_1 = balance::derive_public_key_from_private(privs[0]);

    let sig_2 = sign(privs[1], msg.clone());
    let sig_2_len = sig_2.len() as u8;
    witness.push(sig_2_len);
    let pk_2 = balance::derive_public_key_from_private(privs[1]);
    witness.extend(sig_2.iter());

    let remaining_script = create_multisig_script(vec![pk_1.to_vec(), pk_2.to_vec()]);
    let raw_script_len = remaining_script.len() as u8;
    witness.push(raw_script_len);
    witness.extend(remaining_script.iter());
    return witness;
}

// Given arrays of inputs, outputs, and witnesses, assemble the complete
// transaction and serialize it for broadcast. Return bytes as hex-encoded string
// suitable to broadcast with Bitcoin Core RPC.
// https://en.bitcoin.it/wiki/Protocol_documentation#tx
fn assemble_transaction(
    inputs: Vec<Vec<u8>>,
    outputs: Vec<Utxo>,
    witnesses: Vec<Vec<u8>>,
    is_p2sh: bool,
) -> Vec<u8> {
    let mut transaction: Vec<u8> = vec![];
    let v1 = 2u32.to_le_bytes();
    // println!("\n======================\n");
    // println!("Version: {:?}", hex::encode(v1.clone()));
    transaction.extend(v1.iter());
    transaction.push(0x00);
    transaction.push(0x01); // for witness
    // println!("Marker and Flag: {:?}", hex::encode([0x00, 0x01]));
    transaction.push(0x01); // num ins
    // println!("Number of Inputs: {:?}", hex::encode([0x01]));
    let first_only_input = &inputs[0];
    transaction.extend(first_only_input.iter()); // first input ser
    // println!("First Input: {:?}", hex::encode(first_only_input));
    transaction.push(0x02); // num outs 
    // println!("Number of Outputs: {:?}", hex::encode([0x02]));
    for output in outputs {
        let out_ser = output_from_options(&output.script_pubkey, output.amount);
        transaction.extend(out_ser.iter());
    }
    if is_p2sh {
        transaction.push(0x04);
    } else {
        transaction.push(0x02);
    }
    // println!("Number of Witness Items: {:?}", hex::encode([0x02]));
    // i hope this works :0
    for witness in witnesses {
        // let witness_len = witness.len() as u8;
        // println!("Witness: {:?}", hex::encode(witness.clone()));
        transaction.extend(witness.iter());
    }
    let locktime = 0x00000000u32.to_be_bytes();
    // println!("Locktime: {:?}", hex::encode(locktime.clone()));
    transaction.extend(locktime.iter());

    return transaction;
}

// Given arrays of inputs and outputs (no witnesses!) compute the txid.
// Return the 32 byte txid as a *reversed* hex-encoded string.
// https://developer.bitcoin.org/reference/transactions.html#raw-transaction-format
fn get_txid(inputs: Vec<Vec<u8>>, outputs: Vec<Utxo>) -> [u8; 32] {
    let mut preimage: Vec<u8> = vec![];
    let v1 = 2u32.to_le_bytes();
    preimage.extend(v1.iter());
    let input_len = inputs.len() as u8;
    preimage.push(input_len);
    for input in inputs {
        preimage.extend(input.iter());
    }
    let output_len = outputs.len() as u8;
    preimage.push(output_len);
    for output in outputs {
        let ser_out = output_from_options(&output.script_pubkey, output.amount);
        preimage.extend(ser_out.iter());
    }
    let lock_time = 0x00000000u32.to_be_bytes();
    preimage.extend(lock_time.iter());
    let mut internal = double_hash(preimage);
    internal.reverse();
    let txid_try: Result<[u8; 32], _> = internal.try_into();
    let txid = txid_try.unwrap();
    return txid
}

// Spend a p2wpkh utxo to a 2 of 2 multisig p2wsh and return the (txid, transaction) tupple
pub fn spend_p2wpkh(wallet_state: &WalletState) -> Result<([u8; 32], Vec<u8>), SpendError> {
    // FEE = 1000
    // AMT = 1000000
    // Choose an unspent coin worth more than 0.01 BTC
    let utxos = &wallet_state.utxos;
    let mut selected_outpoint: Option<&OutgoingTx> = None;
    for utxo in utxos {
        if utxo.sats > 1_000_000 {
            selected_outpoint = Some(utxo);
            break;
        }
    }

    // Create the input from the utxo
    let outpoint = selected_outpoint.unwrap();
    // println!("Input SPK: {:?}", hex::encode(outpoint.spk.clone()));
    // Reverse the txid hash so it's little-endian
    let input_external_txid = &outpoint.pointer;
    let mut input_external_txid_bytes = hex::decode(input_external_txid).expect("Could not decode Input TXID.");
    input_external_txid_bytes.reverse();
    let attempt_internal: Result<[u8; 32], _> = input_external_txid_bytes.try_into();
    let internal_repr = attempt_internal.expect("Could not fit TX hash to 32 bytes");
    let input = input_from_utxo(&internal_repr, outpoint.out_point_index);

    // Compute destination output script and output
    let pk1 = &wallet_state.public_keys[0];
    let pk2 = &wallet_state.public_keys[1];
    let raw_script = create_multisig_script(vec![pk1.to_vec(), pk2.to_vec()]);
    let multisig_spk = get_p2wsh_program(&raw_script, None);

    // Compute change output script and output
    let change = (outpoint.sats - 1_000_000 - 1_000) as u32;
    let change_utxo_output =  Utxo { script_pubkey: outpoint.spk.clone(), amount: change as u32 };

    let c_hash_output = Outpoint { txid: internal_repr, index: outpoint.out_point_index };
    let txo = Utxo { script_pubkey: outpoint.spk.clone(), amount: change as u32 };
    let scriptcode = get_p2wpkh_scriptcode(txo);

    let commited_outputs = vec![Utxo { script_pubkey: multisig_spk.clone(), amount: 1_000_000 },
                                                    change_utxo_output];
    
    let commit_hash = get_commitment_hash(c_hash_output, &scriptcode, outpoint.sats as u32, commited_outputs);

    // Fetch the private key we need to sign with
    let mut index_to_take: usize = 0;
    for (index, script) in wallet_state.witness_programs.iter().enumerate() {
        if outpoint.spk.eq(script) { break; }
        index_to_take = index_to_take + 1;
    }
    let priv_key = wallet_state.private_keys[index_to_take];
    let found_pk = balance::derive_public_key_from_private(&priv_key.clone());
    // println!("Found Private Key: {:?}", hex::encode(priv_key.clone()));
    // println!("Found Public Key: {:?}", hex::encode(found_pk.clone()));
    // println!("Confirming Witness Pubkey Matches: {:?}", hex::encode(balance::get_p2wpkh_program(found_pk)));

    // Sign!
    let p2pkh_witness = get_p2wpkh_witness(&priv_key, commit_hash);

    // Assemble
    let multisig_utxo_output = Utxo { script_pubkey: multisig_spk.clone(), amount: 1_000_000 };
    let change_utxo_output =  Utxo { script_pubkey: outpoint.spk.clone(), amount: change as u32 };
    let rawtx = assemble_transaction(vec![input], vec![multisig_utxo_output, change_utxo_output], vec![p2pkh_witness], false);
    // println!("Raw TX: {:?}\n\n", hex::encode(rawtx.clone()));
    
    // Reserialize without witness data and double-SHA256 to get the txid
    let input = input_from_utxo(&internal_repr, outpoint.out_point_index);
    let multisig_utxo_output = Utxo { script_pubkey: multisig_spk.clone(), amount: 1_000_000 };
    let change_utxo_output =  Utxo { script_pubkey: outpoint.spk.clone(), amount: change as u32 };
    let commited_outputs = vec![multisig_utxo_output, change_utxo_output];
    let txid = get_txid(vec![input], commited_outputs);
    // println!("{:?}", hex::encode(txid));
    // For debugging you can use RPC `testmempoolaccept ["<final hex>"]` here

    // return txid, final-tx
    return Ok((txid, rawtx));
 
}

// Spend a 2-of-2 multisig p2wsh utxo and return the transaction
pub fn spend_p2wsh(wallet_state: &WalletState, txid: [u8; 32]) -> Result<([u8; 32], Vec<u8>), SpendError> {
    // COIN_VALUE = 1000000
    // FEE = 1000
    // AMT = 0
    // Create the input from the utxo
    let pk1 = &wallet_state.public_keys[0];
    let pk2 = &wallet_state.public_keys[1];
    let raw_script = create_multisig_script(vec![pk1.to_vec(), pk2.to_vec()]);
    let multisig_spk = get_p2wsh_program(&raw_script, None);
    // Reverse the txid hash so it's little-endian
    let mut input_external_txid_bytes = txid;
    input_external_txid_bytes.reverse();
    let attempt_internal: Result<[u8; 32], _> = input_external_txid_bytes.try_into();
    let internal_repr = attempt_internal.expect("Could not fit TX hash to 32 bytes");

    let tx_input = input_from_utxo(&internal_repr, 0);

    // Compute destination output script and output
    let mut op_return_spk = vec![];
    op_return_spk.push(0x6a); // OP_RETURN
    op_return_spk.push(0x03);
    op_return_spk.push(0x72);
    op_return_spk.push(0x4f);
    op_return_spk.push(0x42);
    let op_return_utxo = Utxo { script_pubkey: op_return_spk, amount: 0 };

    // Compute change output script and output
    let change_spk = balance::get_p2wpkh_program(pk1.clone());
    let change_utxo = Utxo { script_pubkey: change_spk.to_vec(), amount: 1_000_000 - 1_000 };

    // Get the message to sign
    let mut scriptcode: Vec<u8> = vec![];
    let output_l = raw_script.len() as u8;
    scriptcode.push(output_l);
    scriptcode.extend(raw_script.clone().iter());
    let commitment = get_commitment_hash(Outpoint { txid: internal_repr, index: 0 }, &scriptcode, 1_000_000, vec![op_return_utxo, change_utxo]);
    // Sign!
    let priv_key_1 = wallet_state.private_keys[0];
    let priv_key_2 = wallet_state.private_keys[1];
    let witness = get_p2wsh_witness(vec![&priv_key_1, &priv_key_2], commitment);
    // Assemble
    let mut op_return_spk = vec![];
    op_return_spk.push(0x6a); // OP_RETURN
    op_return_spk.push(0x03);
    op_return_spk.push(0x72);
    op_return_spk.push(0x4f);
    op_return_spk.push(0x42);
    let op_return_utxo = Utxo { script_pubkey: op_return_spk, amount: 0 };
    let change_spk = balance::get_p2wpkh_program(pk1.clone());
    let change_utxo = Utxo { script_pubkey: change_spk.to_vec(), amount: 1_000_000 - 1_000 };

    let rawtx = assemble_transaction(vec![tx_input], vec![op_return_utxo, change_utxo], vec![witness], true);
    // For debugging you can use RPC `testmempoolaccept ["<final hex>"]` here
    // println!("{}", hex::encode(rawtx.clone()));
    // return txid final-tx
    let tx_input = input_from_utxo(&internal_repr, 0);
    let mut op_return_spk = vec![];
    op_return_spk.push(0x6a); // OP_RETURN
    op_return_spk.push(0x03);
    op_return_spk.push(0x72);
    op_return_spk.push(0x4f);
    op_return_spk.push(0x42);
    let op_return_utxo = Utxo { script_pubkey: op_return_spk, amount: 0 };
    let change_spk = balance::get_p2wpkh_program(pk1.clone());
    let change_utxo = Utxo { script_pubkey: change_spk.to_vec(), amount: 1_000_000 - 1_000 };

    let txid = get_txid(vec![tx_input], vec![op_return_utxo, change_utxo]);
    // println!("{}", hex::encode(txid));
    Ok((txid, rawtx))
}
```
