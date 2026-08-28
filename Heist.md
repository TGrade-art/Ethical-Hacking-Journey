# Heist

**Platform:** TryHackMe  
**Date:** 2026-07-31  
**Difficulty:** Medium  
**Category:** Web3 / Blockchain / Smart Contract Exploitation  
**Room URL:** [https://tryhackme.com/room/hfb1heist](https://tryhackme.com/room/hfb1heist)  
**Status:** ✅ Completed

---

## 📌 What Was This Room About?

This room was about exploiting a vulnerable Ethereum smart contract belonging to Cipher.

The goal wasn't just to mess with the contract — we had to actually **drain the ETH from its treasury**, which would cut off the funding for the Phantom Node Botnet.

The interesting part was that the contract had a weakness in its ownership system. By exploiting `changeOwnership()`, I could take control of the contract and then use `withdraw()` to empty its balance.

Once the treasury was drained, `isSolved()` changed from `false` to `true and the room was complete.

---

## 🔍 Reconnaissance

As always, I started by looking at what TryHackMe gave me.

The challenge provided an RPC endpoint, an API endpoint, and a wallet that I could use to interact with the blockchain.

I first set up the environment variables:

```bash
RPC_URL=http://10.129.178.132:8545
API_URL=http://10.129.178.132

PRIVATE_KEY=$(curl -s $API_URL/challenge | jq -r ".player_wallet.private_key")
CONTRACT_ADDRESS=$(curl -s $API_URL/challenge | jq -r ".contract_address")
PLAYER_ADDRESS=$(curl -s $API_URL/challenge | jq -r ".player_wallet.address")
```

I then checked the current state of the contract:

```bash
is_solved=$(cast call $CONTRACT_ADDRESS "isSolved()(bool)" --rpc-url $RPC_URL)
echo "Check if is solved: $is_solved"
```

The result was:

```text
Check if is solved: false
```

So obviously, I had some work to do.

---

# 🧭 Steps I Took

## Step 1 — Checking the Smart Contract

The first thing I wanted to understand was what the contract actually did.

The challenge gave me the contract address and RPC endpoint, so instead of treating it like a normal web application, I started thinking about it as an actual blockchain target.

The important thing here was identifying the functions available on the contract and figuring out which ones could modify its state.

The goal was clearly to get:

```text
isSolved() → true
```

But I didn't want to simply force the solved state. I needed to understand **what the challenge expected me to exploit**.

---

## Step 2 — Getting `cast` Working

The obvious tool for interacting with an Ethereum contract from the command line is `cast`, which is part of Foundry.

I tried running it:

```bash
cast
```

But the AttackBox didn't recognize the command.

```text
command not found
```

So I tried getting Foundry installed.

```bash
foundryup
```

After sourcing my shell configuration:

```bash
source ~/.bashrc
```

I tried again.

Unfortunately, I got:

```text
cannot execute binary file
```

So that wasn't going to work normally.

I ended up using an alternative way of getting access to `cast` so I could continue interacting with the contract.

---

## Step 3 — Finding the Contract Vulnerability

This was the important part.

After examining the contract's functionality, I found that it had an ownership mechanism involving:

```text
changeOwnership()
```

The problem was that the function didn't properly protect who was allowed to change the owner.

That meant I could interact with the contract and change its owner to my player wallet.

This was the actual vulnerability.

### Why this matters

Ownership is extremely important in smart contracts.

If a contract has functions that are supposed to be restricted to the owner, but the ownership mechanism itself can be abused, an attacker can potentially gain access to all of those privileged functions.

And in this case, one of those functions was:

```text
withdraw()
```

That was basically the jackpot.

---

## Step 4 — Taking Ownership

Once I understood the vulnerability, I used `cast` to call the vulnerable ownership function and set the contract owner to my player address.

The important idea was:

```text
Attacker
   ↓
changeOwnership()
   ↓
Attacker becomes owner
```

I then had the permissions required to use the contract's owner-only functionality.

This was the point where the vulnerability went from being theoretical to actually giving me control over the contract.

---

## Step 5 — Draining the Treasury

Now that I had ownership, I could use the privileged `withdraw()` functionality.

The contract contained ETH in its treasury, and the objective of the room was to drain it.

So the attack chain was basically:

```text
Find vulnerable contract
        ↓
Exploit changeOwnership()
        ↓
Become contract owner
        ↓
Call withdraw()
        ↓
Drain contract's ETH
        ↓
isSolved() = true
```

This is what makes this room pretty cool compared to a normal web exploit.

Instead of getting a shell or reading a flag file, I was manipulating **actual smart-contract state on an Ethereum-compatible blockchain**.

---

## Step 6 — Checking the Result

After carrying out the exploit, I checked the contract again:

```bash
is_solved=$(cast call $CONTRACT_ADDRESS "isSolved()(bool)" --rpc-url $RPC_URL)
echo "Check if is solved: $is_solved"
```

Before the exploit:

```text
Check if is solved: false
```

After draining the treasury:

```text
Check if is solved: true
```

That confirmed that the exploit had worked.

---

# 🛠 Commands Used

|Command / Tool|What It Does|
|---|---|
|`curl`|Retrieves the challenge information from the API.|
|`jq`|Extracts values such as the wallet private key, contract address and player address from JSON.|
|`cast call`|Reads data from a smart contract without changing its state.|
|`cast send`|Sends a transaction to a smart contract to modify its state.|
|`changeOwnership()`|Vulnerable contract function that allowed ownership to be changed.|
|`withdraw()`|Privileged function used to withdraw the contract's ETH.|
|`isSolved()`|Checks whether the challenge has been successfully completed.|
|`source ~/.bashrc`|Reloads the shell configuration after modifying the environment.|

---

## 🚩 Flags Found

|Flag|Value|
|---|---|
|Room Flag|No traditional flag was displayed|
|Completion State|`isSolved() = true`|

The important result was the contract's state changing from:

```text
false
```

to:

```text
true
```

---

## 💡 What I Learned

- I learned that **smart contracts are basically programs running on a blockchain**, so vulnerabilities in their logic can have very different consequences from normal web vulnerabilities.
    
- I learned why **access control is extremely important in smart contracts**. If an attacker can abuse something like `changeOwnership()`, they can potentially gain access to functions that were supposed to be restricted.
    
- I got more experience using **Foundry and `cast`** to interact directly with Ethereum smart contracts instead of using a normal web interface.
    
- I also learned that when looking at a smart contract, I shouldn't just look for obvious bugs. I need to think about **what happens when different functions are chained together**. In this case, `changeOwnership()` by itself was the important weakness, but `withdraw()` was what made that weakness dangerous.
    
- Most importantly, I learned that a blockchain CTF can have a completely different attack flow from a normal Linux or web box. There wasn't a `/etc/passwd`, shell or normal flag file to hunt for — the target was the **contract's state and ETH balance**.
    

---

## ❓ What Confused Me / What I Want to Research Next

- The `foundryup` installation problem confused me because the command existed but the binary couldn't execute. I want to understand exactly what caused the architecture/compatibility issue.
    
- I want to learn more Solidity so I can actually read vulnerable smart contracts myself instead of relying on the function names to tell me what is happening.
    
- I also want to understand Ethereum transactions better, especially how `cast send`, gas, transaction signing and contract state changes all fit together.
    
- The biggest thing I want to research next is **smart-contract access control** and common vulnerabilities involving ownership.
    

---

## 🔗 Linked Notes

- [[Web3 Hacking]]
    
- [[Smart Contracts]]
    
- [[Solidity]]
    
- [[Ethereum]]
    
- [[Foundry]]
    
- [[Cast CLI]]
    
- [[Blockchain Security]]
    
- [[Access Control]]
    

---

## 📎 Resources Used

- TryHackMe — **Heist**
    
- Foundry / `cast` documentation
    
- Smart-contract analysis during the challenge
    

---

_Report written in [[TryHackMe Vault]] — part of my ethical hacking journey 🛡️_