---
title: "Optimization Algorithm"
date: "Jan 31 2024"
description: "Simple revenue optimization for Bitcoin miners."
---

## Context

This was the week 3 exercise for the **Chaincode: Start Your Career in FOSS** program. Compared to last week, I wrote some pretty good code in this one. 
The goal here is to assemble a valid block from a mempool of pending transactions. "Valid" here is actually pretty complicated because transactions are allowed to 
spend the output of other transactions as long as they are both included in the block. This relationship is referred to as child-parent where a child spends a parent. 
Every transaction has a *weight* and pays a *fee*. The *weight* is how "big" a transaction is.
The rules are as follows:

- The total weight of transactions in a block must not exceed 4,000,000 weight.
- A transaction may only appear in a block if all of its parents appear earlier in the block.
- A transaction must not appear more than once in the block.
- A transaction may have zero, one or many parents in the mempool. It may also have zero, one or many children in the mempool.


## Optimization Metric

Since a block has a weight limit, a block should take the transactions that maximize *Fee* divided by *Weight* for the block. A transaction that has a high ratio of fees relative to its size
**must also** contain all of its ancestors. To avoid taking transactions that have high fees themselves but have cheap ancestors, the algorithm must recursively calculate the 
*Fee* divided by *Weight*. Now the algorithm can sort on this metric and start taking transactions. When the algorithm is adding transaction to a block, the algorithm has to include all of the parents for a given transaction before it, and should accurately calculate the *Weight* added in the case some ancestors are already in the block. 

```rust
use std::{collections::{HashMap, HashSet}, error::Error};
use std::fs::File;
use std::io::Write;

const FILE_PATH: &str = "../mempool.csv";
const MAX_WEIGHT: f32 = 4_000_000.0;

#[derive(Clone,Debug,serde::Deserialize)]
struct Transaction {
    txid: String,
    fee: f32,
    weight: f32,
    parents: Option<String>,
}

type HeurtisticMap = (String, f32);

/**
 * This algorithm assumes that the maximum of child(fee / weight) + (average(fee / weight)) / 2 for parent transacitons
 * is revenue maximizing. The algorithm computes that metric for each child and its parents and sorts on that transaction. 
 * I assume this is a decent approach because if a transaction is to be included, all of its parents need to be included, 
 * so each transaction should be counted equally and given a final (fee / weight) score. 
 */
fn set_all_heuristics(transactions: &Vec<Transaction>, map: &mut Vec<HeurtisticMap>, parents_of: &HashMap<String, Vec<String>>, fees: &HashMap<String, f32>, weights: &HashMap<String, f32>) {
    for transaction in transactions {
        let heuristic = compute_heuristic(transaction.txid.clone(), parents_of, fees, weights);
        map.push((transaction.txid.clone(), heuristic));
    }
}

/**
 * Recursively calculate the heuristic
 */
fn compute_heuristic(child: String, parents_of: &HashMap<String, Vec<String>>, fees: &HashMap<String, f32>, weights: &HashMap<String, f32>) -> f32 {
    let parents = parents_of.get(&child);
    match parents {
        Some(parents) => {
            let length_parents = parents.len() as f32;
            let mut parent_weights = 0.0;
            for parent in parents {
                parent_weights += compute_heuristic(parent.to_string(), parents_of, fees, weights);
            }
            let average = parent_weights / length_parents;
            let child_heuristic = *fees.get(&child).expect("All fees should be populated.") / *weights.get(&child).expect("All weights should be populated.");
            return (average + child_heuristic) / 2.0;
            
        },
        None => {
            return *fees.get(&child).expect("All fees should be populated.") / *weights.get(&child).expect("All weights should be populated.")
        },
    }
}
/**
 * Calculate the weight of a transaction to add it to the total. It is assumed that this transaction has a high (fee / weight) metric, 
 * so all parents should be included, and, subsequently, their ancestors.
 */
fn weight_of(child: String, parents_of: &HashMap<String, Vec<String>>, weights: &HashMap<String, f32>, inclusions: &HashSet<String>) -> f32 {
    match parents_of.get(&child) {
        Some(parents) => {
            let mut parent_weights = 0.0;
            let mut child_weight: f32 = 0.0;
            if !inclusions.contains(&child) {
                child_weight = *weights.get(&child).expect("All weights should be populated.");
            } 
            for parent in parents {
                parent_weights += weight_of(parent.to_string(), parents_of, weights, &inclusions);
            }
            return parent_weights + child_weight
            
        },
        None => {
            if !inclusions.contains(&child) {
                return *weights.get(&child).expect("All weights should be populated.")
            } else {
                0.0
            }
        },
    }
}

/**
 * Return all the ancestors of a child, unless they are already in the block.
 */
fn all_ancestors(child: String, parents_of: &HashMap<String, Vec<String>>, inclusions: &HashSet<String>) -> Vec<String> {
    match parents_of.get(&child) {
        Some(parents) => {
            let mut ancestors = vec![];
            if !inclusions.contains(&child) {
                ancestors.push(child);
            } // push itself and all parents
            for parent in parents {
                let all_ancestors = all_ancestors(parent.to_string(), parents_of, inclusions);
                for ancestor in all_ancestors {
                    if !inclusions.contains(&ancestor) {
                        ancestors.push(ancestor);
                    }
                }
            }
            return ancestors
        },
        None => {
            if !inclusions.contains(&child) {
                vec![child]
            } else {
                vec![]
            }
        },
    }
}

pub fn build_block() -> Result<(), Box<dyn Error>> {
    let mut rdr = csv::ReaderBuilder::new().has_headers(false).from_path(FILE_PATH)?;
    let mut transactions: Vec<Transaction> = vec![];
    let mut heuristic_map: Vec<HeurtisticMap> = Vec::new();
    let mut parents_of: HashMap<String, Vec<String>> = HashMap::new();
    let mut weights: HashMap<String, f32> = HashMap::new();
    let mut fees: HashMap<String, f32> = HashMap::new();
    let mut block: Vec<String> = Vec::new();

    // O(N) pass
    for record in rdr.deserialize() {
        let transaction: Transaction = record?;
        transactions.push(transaction.clone());
        weights.insert(transaction.txid.clone(), transaction.weight);
        fees.insert(transaction.txid.clone(), transaction.fee);
        match transaction.parents {
            Some(parents) => {
                let parents: Vec<String> = parents.split(";").map(String::from).collect();
                parents_of.insert(transaction.txid.clone(), parents);
            },
            None => {
                continue;
            },
        }
    }

    // O(??) pass, probably big lol. If each child has X parents it is O(N*X)?
    set_all_heuristics(&transactions, &mut heuristic_map, &parents_of, &fees, &weights);
    heuristic_map.sort_by(|(_, a), (_, b)| b.partial_cmp(a).expect("Error occured comparing floating points"));

    let mut inclusions: HashSet<String> = HashSet::new();
    let mut block_weight: f32 = 0.0;

    for (id, _) in heuristic_map.iter() {
        // do not include a transaction if it is already in the block
        if inclusions.contains(id) { 
            continue; 
        } 
        
        // to speed this up, if I just get the ancestors first I can calculate the next weight in near constant time
        let next_weight = weight_of(id.to_string(), &parents_of, &weights, &inclusions); // we must include ALL ancestors and count their weights, unless they are already in the block
        let next_block_weight = next_weight + block_weight; 
        if next_block_weight > MAX_WEIGHT { break; } // we are done because no more transactions will fit

        block_weight += next_weight; // we added the weight of the transaction and all ancestors, excluding the weight of transactions already present
        // O(N*X)?
        let mut ancestors = all_ancestors(id.to_string(), &parents_of, &inclusions); // we need to actually fetch the ancestors, unless they are already in the block
        ancestors.reverse(); // place the parents first
        block.extend_from_slice(&ancestors);

        for ancestor in ancestors {
            inclusions.insert(ancestor);
        }

    }
    println!("\nLength of inclusion set {:?}", inclusions.len());
    println!("Length of block {:?}", block.clone().len());
    println!("Block weight: {:?}", calc_weight(block.clone(), &weights)); 
    println!("Total Fee: {:?}\n", calc_fee(block.clone(), &fees));
    let is_valid = validate_block(block.clone(), &weights, &parents_of, &inclusions);
    println!("Block is valid, writing to file: {:?}", is_valid);
    
    if is_valid { write_block_to_file(block); }
    Ok(())
}


// block validation 
fn calc_fee(block: Vec<String>, fees: &HashMap<String, f32>) -> f32 {
    let mut total: f32 = 0.0;
    for tx in block {
        total += *fees.get(&tx).expect("All fees should be populated.");
    }
    return total
}

fn calc_weight(block: Vec<String>, weights: &HashMap<String, f32>) -> f32 {
    let mut total: f32 = 0.0;
    for tx in block {
        total += *weights.get(&tx).expect("All weights should be populated.");
    }
    return total
}

fn has_duplicates(block: Vec<String>) -> bool {
    let mut set = HashSet::new();
    for s in block {
        if !set.insert(s) {
            return true;
        }
    }
    false
}

fn parents_after_children(block: Vec<String>, parents_of: &HashMap<String, Vec<String>>) -> bool {
    let mut tx_set: HashSet<String> = HashSet::new();
    for tx in block {
        tx_set.insert(tx.clone());
        match parents_of.get(&tx) {
            Some(parents) => {
                for parent in parents {
                    if !tx_set.contains(parent) {
                        return true;
                    }
                }
            },
            None => continue,
        }
    }
    return false
}

fn parents_included(block: Vec<String>, parents_of: &HashMap<String, Vec<String>>, inclusions: &HashSet<String>) -> bool {
    for tx in block {
        match parents_of.get(&tx) {
            Some(parents) => {
                for parent in parents {
                    if !inclusions.contains(parent) {
                        println!("Some parent is not in a block: {:?}", parent);
                        return false;
                    }
                }
            },
            None => continue,
        }
    }
    return true
}

fn validate_block(block: Vec<String>, weights: &HashMap<String, f32>, parents_of: &HashMap<String, Vec<String>>, inclusions: &HashSet<String>) -> bool {
    if calc_weight(block.clone(), weights) > MAX_WEIGHT {
        println!("Weight is exceeded.");
        return false;
    }
    if has_duplicates(block.clone()) {
        println!("Block not valid because there exists duplicates.");
        return false;
    }
    if parents_after_children(block.clone(), parents_of) {
        println!("Block not valid because a child occured before a parent.");
        return false;
    }
    if !parents_included(block.clone(), parents_of, inclusions) {
        println!("Block not valid because some parents were not included.");
        return false;
    }
    return true;
}

fn write_block_to_file(block: Vec<String>) {
    let file_path = "block.txt";
    let mut file = File::create(file_path).expect("Could not create or write file.");
    for tx in &block {
        writeln!(file, "{}", tx).expect("Could not write transaction");
    }
}
```