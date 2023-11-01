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
- **Implications**: The players that entered the same raffle as the attacker are victims of this exploit as players cannot get refunded anymore, the owner will not get his fees and the winner will not get his prize amount.

## Recommendation

Make the following change:

```diff
- payable(msg.sender).sendValue(entranceFee);
+ players[playerIndex] = address(0);

- players[playerIndex] = address(0);
+ payable(msg.sender).sendValue(entranceFee);

```
## Proof of Concept for [DoS Attack]

### Overview:
There's a dangerous check that was made in the absence of fallback functions in the contract. That check is in the withdrawFees() function that allows the feeAddress to withdraw fees.

### Actors:
- **Attacker**: the malicious EOA/contract to perform the DoS attack.
- **Victim**: owner of the feeAddress, so owner most probably.
- **Protocol**: The raffle contract itself.

### Exploit Scenario:
- **Initial State**: winner has been selected and the victim is ready to call withdrawFees().
- **Step 1**: the attacker decides to send a bit of ether, just enough to make address(this).balance different from uint256(totalFees).
- **Step 2**: solidity```require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");``` fails when the victim calls withdrawFees().
- **Outcome**: Victim cannot take its profits.
- **Implications**: Victim will have its funds locked in the contract

## Recommendation

Make the following change:

```diff
+ receive() external payable {
  require(msg.sender == address(0), "Never recive funds outside of the enterRaffle() function"
};
-

```
