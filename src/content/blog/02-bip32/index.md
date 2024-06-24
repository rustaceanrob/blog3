---
title: "Bitcoin Improvement Proposal 32"
description: "The internals of a bitcoin wallet, built from scratch in Rust"
date: "Jan 23 2024"
---

## Context
  
This exercise was completed as part of the **Chaincode: Start Your Career in FOSS** program. BIP32 (and BIP39) is a specification for wallet developers that makes wallet recovery as simple as possible for users without security compromises. I built an implementation from scratch with some pretty terrible code that would never come close to a production environment. The output is what matters (:


```rust
extern crate bitcoincore_rpc;
use bitcoincore_rpc::{Auth, Client, RpcApi};
use bs58;
use hex;
use hmac_sha512::HMAC;
use num_bigint::BigUint; use ripemd::digest::{FixedOutput, Update};
use ripemd::digest::crypto_common::KeyInit;
// for modulus math on large numbers
use ripemd::{Ripemd160, Ripemd160Core};
use secp256k1::{Secp256k1, SecretKey, PublicKey};
use sha2::{Sha256, Digest};
use std::error::Error;
use std::hash::Hash;
use std::io::{Read, Write};
use std::ops::Add;
use std::path::PathBuf;
use std::str;

// Provided by administrator
const WALLET_NAME: &str = "wallet_164";
const EXTENDED_PRIVATE_KEY: &str = "tprv8ZgxMBicQKsPdpZe8dbST5dNjj5UkA9MDWR9MKZw4vdHoJ27dXZbWSGBE7BgaURHs3A649nhUsKcDvQbGVHUQRY68jHKEu6XzDz4Kq5pWGa";
const HARDENED_OFFSET: u32 = 2_u32.pow(31);
const CURVE_ORDER: &str = "FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141";

#[derive(Debug)]
struct ExtendedKey {
    pub version_bytes: [u8; 4],
    pub depth: [u8; 1],
    pub finger_print: [u8; 4],
    pub child_number: [u8; 4],
    pub chain_code: [u8; 32],
    pub key: [u8; 33],
}

struct ChildKey {
    pub priv_key: [u8; 32],
    pub chain_code: [u8; 32],
}

pub struct OutgoingTx {
    pub sats: u64,
    pub pointer: String,
}

struct SpendingTx {
    pub prev_out: String,
}

// final wallet state struct
pub struct WalletState {
    pub utxos: Vec<OutgoingTx>,
    witness_programs: Vec<[u8; 22]>,
    public_keys: Vec<[u8; 33]>,
    private_keys: Vec<[u8; 32]>,
}

// Decode a base58 string into an array of bytes
fn base58_decode(base58_string: &str) -> Vec<u8> {
    let base58_alphabet = "123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz";

    // Convert Base58 string to a big integer
    // Convert the integer to bytes
    let full_decoded = bs58::decode(base58_string)
                                                .into_vec()
                                                .expect("Decoding failed.");
    let cs = &full_decoded[full_decoded.len() - 4..];

    // Chop off the 32 checksum bits and return
    let mut decoded = full_decoded.clone();
    decoded.truncate(decoded.len() - 4);

    // BONUS POINTS: Verify the checksum!
    let mut hasher = Sha256::new();
    hasher.write(&decoded.clone()).expect("Hash failed.");
    let first_digest = hasher.finalize_fixed();
    let digest_bytes = first_digest.to_vec();
    let mut hasher = Sha256::new();
    hasher.write(&digest_bytes.clone()).expect("Hash failed.");
    let second_digest = hasher.finalize_fixed();
    let digest_bytes = second_digest.to_vec();
    let challenge = &digest_bytes[0..4];
    assert_eq!(challenge, cs);

    return decoded
}

// Deserialize the extended key bytes and return a JSON object
// https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki#serialization-format
// 4 byte: version bytes (mainnet: 0x0488B21E public, 0x0488ADE4 private; testnet: 0x043587CF public, 0x04358394 private)
// 1 byte: depth: 0x00 for master nodes, 0x01 for level-1 derived keys, ....
// 4 bytes: the fingerprint of the parents key (0x00000000 if master key)
// 4 bytes: child number. This is ser32(i) for i in xi = xpar/i, with xi the key being serialized. (0x00000000 if master key)
// 32 bytes: the chain code
// 33 bytes: the public key or private key data (serP(K) for public keys, 0x00 || ser256(k) for private keys)
fn deserialize_key(bytes: Vec<u8>) -> ExtendedKey {  
    let version_bytes: Result<[u8; 4], _> = bytes[0..4].try_into();
    let depth: Result<[u8; 1], _> = bytes[4..5].try_into();
    let finger_print: Result<[u8; 4], _> = bytes[5..9].try_into();
    let child_number: Result<[u8; 4], _> = bytes[9..13].try_into();
    let chain_code: Result<[u8; 32], _> = bytes[13..45].try_into();
    let key: Result<[u8; 33], _> = bytes[45..].try_into();

    return ExtendedKey { version_bytes: version_bytes.unwrap(), 
                            depth: depth.unwrap(), 
                            finger_print: finger_print.unwrap(), 
                            child_number: child_number.unwrap(), 
                            chain_code: chain_code.unwrap(), 
                            key: key.unwrap() }
}

// Derive the secp256k1 compressed public key from a given private key
// BONUS POINTS: Implement ECDSA yourself and multiply your key by the generator point!
// -> [u8; 33]
fn derive_public_key_from_private(key: &[u8; 32]) -> [u8; 33] {
    let curve = Secp256k1::new();
    let sk = SecretKey::from_slice(key)
                                    .expect("Secret key initializatoin failure.");
    let public_key = PublicKey::from_secret_key(&curve, &sk);
    let ser: [u8; 33] = public_key.serialize();
    return ser
}

// Perform a BIP32 parent private key -> child private key operation
// Return a JSON object with "key" and "chaincode" properties as bytes
// https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki#user-content-Private_parent_key_rarr_private_child_key
fn derive_priv_child(key: &[u8; 32], chaincode: &[u8; 32], index: u32) -> ChildKey {
    let is_hardened = index >= HARDENED_OFFSET;
    let ser_i = index.to_be_bytes();
    let hmac_input: Vec<u8>;


    if is_hardened {
        let stack: Vec<u8> = vec![0];
        hmac_input = stack.iter().chain(key.iter().chain(ser_i.iter())).cloned().collect();

    } else {
        let pk = derive_public_key_from_private(key);
        hmac_input = pk.iter().chain(ser_i.iter()).cloned().collect();
    }

    let mac_result = HMAC::mac(hmac_input, chaincode);
    let (child_k, child_chaincode) = mac_result.split_at(32);

    let cc: Result<[u8; 32], _> = child_chaincode.try_into();
    let cc = cc.unwrap();

    let big_child = BigUint::from_bytes_be(&child_k);
    let big_parent = BigUint::from_bytes_be(key);
    let curve_order = hex::decode(CURVE_ORDER).unwrap();
    let big_order = BigUint::from_bytes_be(&curve_order);
    let ck = big_child.add(big_parent).modpow(&BigUint::new(vec![1]), &big_order);
    let big_endian = ck.to_bytes_be();

    // let k: Result<[u8; 32], _> = big_endian.try_into();
    // let k = k.unwrap();

    let mut buffer: [u8; 32] = [0; 32];
    let start_index = 32 - big_endian.len();
    buffer[start_index..].copy_from_slice(&big_endian);

    ChildKey { priv_key: buffer, chain_code: cc }
}

// Given an extended private key and a BIP32 derivation path, compute the child private key found at the last path
// The derivation path is formatted as an array of (index: int, hardened: bool) tuples.
fn get_child_key_at_path(key: [u8; 32], chaincode: [u8; 32], paths: Vec<(u32, bool)>) -> ChildKey {
    let mut final_child: ChildKey = ChildKey { priv_key: key, chain_code: chaincode };

    for (index, is_hardened) in paths {
        let mut offset = index;
        if is_hardened { offset = index + HARDENED_OFFSET }
        final_child = derive_priv_child(&final_child.priv_key, &final_child.chain_code, offset);
    }

    return final_child;
}

// Compute the first N child private keys.
// Return an array of keys encoded as bytes.
fn get_keys_at_child_key_path(child_key: ChildKey, num_keys: u32) -> Vec<[u8; 32]> {

    let mut children: Vec<[u8; 32]> = vec![];

    for index in 0..num_keys {
        let child = derive_priv_child(&child_key.priv_key, &child_key.chain_code, index);
        children.push(child.priv_key);
    }
    
    children
}

// Derive the p2wpkh witness program (aka scriptPubKey) for a given compressed public key.
// Return a bytes array to be compared with the JSON output of Bitcoin Core RPC getblock
// so we can find our received transactions in blocks.
// These are segwit version 0 pay-to-public-key-hash witness programs.
// https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki#user-content-P2WPKH
fn get_p2wpkh_program(pubkey: [u8; 33]) -> [u8; 22] {

    //sha256
    let mut hasher = Sha256::new();
    hasher.write(&pubkey.clone()).expect("Hash failed.");
    let first_digest = hasher.finalize_fixed();
    let digest_bytes = first_digest.to_vec();

    //ripemd160
    let mut hasher = Ripemd160::new();
    hasher.write(&digest_bytes.clone()).unwrap();
    let second_digest = hasher.finalize_fixed();
    let pkh = second_digest.to_vec();

    let version_stack = vec![0, 20];
    let witness_program: Vec<u8> = version_stack.iter().chain(pkh.iter()).cloned().collect();
    let witness_try: Result<[u8; 22], _> = witness_program.try_into();
    let witness = witness_try.unwrap();
    return witness;
    
}

// public function that will be called by `run` here as well as the spend program externally
pub fn recover_wallet_state(extended_private_key: &str, cookie_filepath: &str) -> Result<WalletState, Box<dyn Error>> {
    // Deserialize the provided extended private key
    let ext_key = deserialize_key(base58_decode(extended_private_key));

    // Derive the key and chaincode at the path in the descriptor (`84h/1h/0h/0`)
    // Get the child key at the derivation path
    let path: Vec<(u32, bool)> = vec![(84, true), (1, true), (0, true), (0, false)];
    let master_priv_key_try: Result<[u8; 32], _> = ext_key.key[1..].try_into();
    let master_priv_key = master_priv_key_try.unwrap();
    let account_master_key = get_child_key_at_path(master_priv_key, ext_key.chain_code, path);

    // Compute 2000 private keys from the child key path
    let bag_of_keys = get_keys_at_child_key_path(account_master_key, 2000);
    // For each private key, collect compressed public keys and witness programs
    let mut private_keys = vec![];
    let mut public_keys = vec![];
    let mut witness_programs = vec![];

    for key in bag_of_keys {
        private_keys.push(key);
        let pk = derive_public_key_from_private(&key);
        public_keys.push(pk);
        witness_programs.push(get_p2wpkh_program(pk))
    }

    // Collect outgoing and spending txs from a block scan
    let mut outgoing_txs: Vec<OutgoingTx> = vec![];
    let mut spending_txs: Vec<SpendingTx> = vec![];
    let mut utxos: Vec<OutgoingTx> = vec![];

    // set up bitcoin-core-rpc on signet
    let path = PathBuf::from(cookie_filepath);
    let rpc = Client::new("http://localhost:38332", Auth::CookieFile(path))?;

    // Scan blocks 0 to 300 for transactions
    // Check every tx input (witness) for our own compressed public keys. These are coins we have spent.
    // Check every tx output for our own witness programs. These are coins we have received.
    // Keep track of outputs by their outpoint so we can check if it was spent later by an input
    // Collect outputs that have not been spent into a utxo set
    for block in 0..301 {
        let block_hash = rpc.get_block_hash(block).unwrap();
        let block = rpc.get_block(&block_hash).unwrap();
        let txs = block.txdata;

        for tx in &txs {
            
            // Check every tx input (witness) for our own compressed public keys. These are coins we have spent.
            for input in &tx.input {
                let program = input.witness.to_vec();
                if program.len() > 1 {
                    for pk in &public_keys {
                        if pk.to_vec().eq(&program[1]) {
                            spending_txs.push(SpendingTx { prev_out: input.previous_output.txid.to_string() })
                        }
                    }
                }
            }
            
            // Check every tx output for our own witness programs. These are coins we have received.
            for out in &tx.output {
                for witness in &witness_programs {
                    if out.script_pubkey.as_bytes().eq(&witness.to_vec()) {
                        let value = out.value.to_sat();
                        let txid = tx.txid().to_string();
                        let received_tx = OutgoingTx { sats: value, pointer: txid };
                        outgoing_txs.push(received_tx);                 
                    }
                }            
            }
        }
    }
    // Collect outputs that have not been spent into a utxo set
    for spend in spending_txs {
        outgoing_txs.retain(|x| x.pointer != spend.prev_out);
    }

    utxos = outgoing_txs;

    // Return Wallet State
    Ok(WalletState {
        utxos,
        public_keys,
        private_keys,
        witness_programs,
    })
}

pub fn run(rpc_cookie_filepath: &str) -> Result<(), Box<dyn Error>> {
    let utxos = recover_wallet_state(EXTENDED_PRIVATE_KEY, rpc_cookie_filepath)?;
    let balance: u64 = utxos.utxos.iter().map(|x| x.sats).sum();
    let bitcoin = balance as f64 / 100_000_000.;
    println!("{} {:.8}", WALLET_NAME, bitcoin);
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_path_derive() {
        let b_58_decode = base58_decode("tprv8ZgxMBicQKsPdpZe8dbST5dNjj5UkA9MDWR9MKZw4vdHoJ27dXZbWSGBE7BgaURHs3A649nhUsKcDvQbGVHUQRY68jHKEu6XzDz4Kq5pWGa");
        let ext_priv_key = deserialize_key(b_58_decode);
        let master_priv_key_try: Result<[u8; 32], _> = ext_priv_key.key[1..].try_into();
        let master_priv_key = master_priv_key_try.unwrap();
        let path: Vec<(u32, bool)> = vec![(84, true), (1, true), (0, true), (0, false), (0, false)];
        let first_addr_priv_key = get_child_key_at_path(master_priv_key, ext_priv_key.chain_code, path);
        let first_addr_pk = derive_public_key_from_private(&first_addr_priv_key.priv_key);
        assert_eq!(hex::encode(first_addr_pk), "020147a645bf92b4b36a63285e4ac53636e8b6d5086bd1686092c6387ae80bca9a");
    }

    #[test]
    fn test_one_nonhardened_child_key() {
        let b_58_decode = base58_decode("tprv8ijZZJYvoMqbPnvPdXUMXX8Y8VGoJeVVHqcciuy13T5SpktP5LHzbM38Zqcr4FbTpEBSUi62PpwWg18aPjE69Q6fG6Eyrqi9DnBfay6LYUa");
        let ext_priv_key = deserialize_key(b_58_decode);
        let master_priv_key_try: Result<[u8; 32], _> = ext_priv_key.key[1..].try_into();
        let master_priv_key = master_priv_key_try.unwrap();
        let ck = derive_priv_child(&master_priv_key, &ext_priv_key.chain_code, 0);
        let pk = derive_public_key_from_private(&ck.priv_key);
        assert_eq!(hex::encode(pk), "020147a645bf92b4b36a63285e4ac53636e8b6d5086bd1686092c6387ae80bca9a");
    }

    #[test]
    fn test_first_n() {
        let b_58_decode = base58_decode("tprv8gwhSstZCAiemyBm44HMnUnmvuDpwuPQwnUPQB9Ggx5665BSNowtJCddmpVZBv4GmZEaeUXpyCD3RVvBA96kBwWQXdWp5FisWpgYqw7tVZU");
        let ext_priv_key = deserialize_key(b_58_decode);
        let master_priv_key_try: Result<[u8; 32], _> = ext_priv_key.key[1..].try_into();
        let master_priv_key = master_priv_key_try.unwrap();
        let ck = derive_priv_child(&master_priv_key, &ext_priv_key.chain_code, 0);
        let bag_of_keys = get_keys_at_child_key_path(ck, 3);
        for child in bag_of_keys.iter() {
            let pk: [u8; 33] = derive_public_key_from_private(child);
            println!("{:?}", hex::encode(pk));
        }
    }

    #[test]
    fn test_one_priv_to_pub() {
        let hex_k = "1E99423A4ED27608A15A2616A2B0E9E52CED330AC530EDCC32C8FFC6A526AEDD";
        let k_as_arr = hex::decode(hex_k).unwrap();
        let k_try: Result<[u8; 32], _> = k_as_arr.try_into();
        let k = k_try.unwrap();
        let pk = derive_public_key_from_private(&k);
        assert_eq!(hex::encode(pk).to_uppercase(), "03F028892BAD7ED57D2FB57BF33081D5CFCF6F9ED3D3D7F159C2E2FFF579DC341A");
    }

    #[test]
    fn test_witness_program() {
        let hex_k = "1E99423A4ED27608A15A2616A2B0E9E52CED330AC530EDCC32C8FFC6A526AEDD";
        let k_as_arr = hex::decode(hex_k).unwrap();
        let k_try: Result<[u8; 32], _> = k_as_arr.try_into();
        let k = k_try.unwrap();
        let pk = derive_public_key_from_private(&k);
        let witness = get_p2wpkh_program(pk);
        println!("{:?}", witness);
    }

    #[test]
    fn test_rpc_connection() {
        let path = r"/Users/robertnetzke/Library/Application Support/Bitcoin/signet/.cookie";
        let rpc = Client::new("http://localhost:38332", Auth::CookieFile(path.into())).expect("Connection failed.");
        let h = rpc.get_best_block_hash().expect("Failed to get block hash");
        println!("{:?}", h);
    }

    #[test]
    fn test_len() {
        let ln = [253, 238, 150, 214, 253, 188, 109, 105, 201, 30, 244, 241, 216, 206, 247, 217, 204, 116, 137, 7, 99, 3, 77, 50, 23, 31, 74, 23, 94, 115, 90].len();
        println!("{:?}", ln);
    }
}
```