# Snek Raffle security review by 0xjarix

*********************review -********************* **[[5b9554656d53baa2086ab7c74faf8bdeaf81a8b7](https://github.com/Cyfrin/2024-03-snek-raffle)](https://github.com/Cyfrin/2024-03-snek-raffle)**

### The code was reviewed for a total of 2 hours.
---

# High Severity Finding
## H0 - Bad Logic Imlementation

### Overview:
NFTs of different rarity have the same chance of getting minted
```vyper
rarity: uint256 = random_words[0] % 3
```

### Vulnerability Details
from this line of code: rarity:
```vyper
rarity: uint256 = random_words[0] % 3
```
We can deduce that the chances of getting each of the 3 types of NFTs are equal (33.333...% each), unlike what the documentation stated: 
70% of chance of getting a common NFT
25% of chance of getting a rare NFT
5% of chance of getting a legendary NFT
The winner will have equal chances of getting these NFTs

## Recommendation

Make the following change:

```diff
- rarity: uint256 = random_words[0] % 3
+ percentage: uint256 = random_words[0] % 100
+ rarity: uint256 = 0
+ if percentage < LEGEND_RARITY:
+     rarity = 2
+ elif percentage < RARE_RARITY:
+     rarity = 1
```

## H1 - Denial of Service

### Overview:
The winner can be a malicious contract that has a fallback function that reverts when he's receiving the ETH reward.
```vyper
send(recent_winner, self.balance)
```

### Actors
*Protocol*: snek raffle contract  
*Attacker*: the winner

### Exploit Scenario:
- **Initial State**: The Protocol is deployed, players are entering the raffle, as well as some malicious contracts created by the attacker.
- **Step 1**: request_raffle_winner() is called, the state is CALCULATING.
- **Step 2**: rawFulfillRandomWords() is called by the VRF_COORDINATOR which triggers the call to fulfillRandomWords()
- **Step 3**: The attacker created several malicious contract containing a fallback function that reverts when the contract receives ETH and entered the raffle several times with each of his contracts to increase his chances of winning, eventually one of his contracts was selected to be the winner.
- **Outcome**: The winner which is also the Attacker's malicious contract reverts.
- **Implications**: The protocol is under DoS as the fulfillRandomWords function reverts and the state is still Calculating, no other winners can be requested.


## Recommendation
Make the following change:
We could create a ClaimReward() function that will allow the winners to withdraw their rewards at anytime, that way the protocol cannot be under DoS. We can create a dynamic Array to keep track of the winners and their earnings that would be updated inside the fulfillRandomWords() function and inside the ClaimReward().
