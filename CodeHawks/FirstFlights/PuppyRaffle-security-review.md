# Puppy Raffle security review by 0xjarix

*********************review commit hash -********************* **[22bbbb2c47f3f2b78c1b134590baf41383fd354f](https://github.com/Cyfrin/2023-10-Puppy-Raffle)**

### The code was reviewed for a total of 2 hours.
---


## Proof of Concept for [Reentrancy Attack]

### Overview:
Checks-Effects-Interactions order was not respected in the refund() function allowing the attacker to call refund() till he drains the contract's funds

### Actors:
- **Attacker**: the malicious player who's going to perform the reentrancy attack.
- **Victim**: Any player that entered the raffle before the attacker.
- **Protocol**: The raffle contract itself.

### Exploit Scenario:
- **Initial State**: Players are entering the raffle.
- **Step 1**: Attacker enters the raffle as well.
- **Step 2**: Attacker calls the refund() function with his malicious contract by passing in argument its index using getActivePlayerIndex(address(this)).
- **Step 3**: When refund() is called, the protocol makes the precious error of making an external call to the attacker's contract before updating the list of the players' address
- **Step 4**: The attacker doesn't forget to add a receive() or fallback() function to his malicious contract that will keep on calling the refund() function till he drains the prtocol's contract from all its funds.
- **Outcome**: The protocol will lose all its funds.
- **Implications**: The players that entered the raffle before the attacker are victims of this exploit as players cannot get refunded anyomre, the owner will not get his fees and the winner will not get his prize amount.

## Recommendation

Make the following change:

```diff
- payable(msg.sender).sendValue(entranceFee);

- players[playerIndex] = address(0);
+ players[playerIndex] = address(0);

+ payable(msg.sender).sendValue(entranceFee);

```
