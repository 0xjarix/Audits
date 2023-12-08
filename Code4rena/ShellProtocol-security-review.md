# Shell Protocol List security review by 0xjarix

*********************review code base -********************* **[Shell Protocol's code base](https://github.com/code-423n4/2023-11-shellprotocol/tree/main)**

### The code was reviewed for a total of 1 hour.
---


## H0 [Bad Math Arithmetic in convertDecimals()]

### Overview:
The convertDecimals() function inside the OceanAdapter contract performs bad arithmetic

### Actors:
- **Attacker**: the malicious user.
- **Victim**: Santa.
- **Protocol**: The SantasList contract itself.

### Exploit Scenario:
- **Initial State**: The Protocol is already deployed and the Victim is calling the checkList() function a few times for some addresses.
- **Step 1**: the Victim calls checkList() by passing as a 1st argument the address of a person that turns out to be the Attacker and as a 2nd argument the status NAUGHTY.
- **Step 2**: The Attacker calls getNaughtyOrNiceOnce() by passing as argument his address and gets as a return value the status NAUGHTY.
- **Step 3**: the Attacker calls checkList() by passing as a 1st argument his address and as a 2nd argument the status NICE.
- **Step 4**: the Victim calls checkTwice() by passing as a 1st argument the address of the Attacker and as a 2nd argument the status NAUGHTY.
- **Outcome**: checkTwice() reverts with SantasList__SecondCheckDoesntMatchFirst() error.
- **Implications**: If all the people would call checkList() right after santa to change their status to be the opposite of the return value of getNaughtyOrNiceOnce(), christmas will be ruined as no checkTwice() would revert everytime and no one would be elligible for a present.

## Recommendation

Make the following change:

```diff
- function checkList(address person, Status status) external {
+ function checkList(address person, Status status) external onlySanta {

```
