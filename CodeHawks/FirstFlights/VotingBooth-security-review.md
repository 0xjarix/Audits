# Voting Booth security review by 0xjarix

*********************review commit hash -********************* **[5b9554656d53baa2086ab7c74faf8bdeaf81a8b7](https://github.com/Cyfrin/2023-12-Voting-Booth)**

### The code was reviewed for a total of 2 hours.
---


## Proof of Concept for []

### Overview:
VotersFor rewards is less than it should be and funds are locked forever due to bad logic implementation.

### Actors:
- **Victim**: 
- **Protocol**: 

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
