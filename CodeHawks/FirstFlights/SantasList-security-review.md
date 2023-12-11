# Santa's List security review by 0xjarix

*********************review commit hash -********************* **[e3370783aeda4b41e0054cf1febe75020b0beaae](https://github.com/Cyfrin/2023-11-Santas-List)**

### The code was reviewed for a total of 1 hour.
---


## H0 [Bad Access Control for checkList()]

### Overview:
According to the documentation, checkList() should only be called by santa, hence it is missing the onlySanta() modifier. 

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

## H1 [Bad ALogic Implementation When Buying a Present for Someone Else]

### Overview:
The function buyPresent() is badly implemented as it burns santaTokens in an unsafe manner (could revert a transaction) from the presentReceiver instead of the msg.sender and sends the NFT to the msg.sender instead of the presentReceiver.

### Actors:
- **Attacker**: The presentReceiver and the caller of buyPresent() whose Status is NAUGHTY or NICE.
- **Victim**: Everyone else, santa, the protocol.
- **Protocol**: The SantasList contract itself.

### Exploit Scenario:
- **Initial State**: The Protocol is already deployed and everyone already collected their presents.
- **Step 1**: the Attacker who has a santaToken balance of 0 calls buyPresent() by passing as argument his address.
- **Outcome**: the balance of the attacker reaches type(uint256).max because of the underflow inside the _burn() function from the solmate library that uses the unchecked() box for optimization purposes. That function was called by the burn() function of SantaToken that was called by buyPresent().
- **Implications**: This vulnerability makes it possible for pretty much everyone to print money just by calling a function. The whole protocol is broken because the amount of tokens being maxed out by the attacker is not the same as the number of tokens in the system. 

## Recommendation

Make the following change:

```diff
-
+        require(balanceOf(presentReceiver) >= 1e18, "Not enough tokens in the balance");
-        i_santaToken.burn(presentReceiver);
+        i_santaToken.burn(msg.sender);
-        _mintAndIncrement();
+        _safeMint(presentReceiver, s_tokenCounter++);

```

## H2 [Bad Logic Implementation for collectPresent()]

### Overview:
poorly written check allowing NICE people to buy an infinite amount of NFT via collectPresent() 

### Actors:
- **Attacker**: the malicious NICE person.
- **Victim**: SantasList.
- **Protocol**: The SantasList contract itself.

### Exploit Scenario:
- **Initial State**: The Protocol is already deployed and the people are calling the collectPresent() function.
- **Step 1**: The Attacker calls collectPresent() and gets his NFT.
- **Step 2**: The Attacker calls collectPresent() and gets his NFT.
- **Step 3**: The Attacker calls collectPresent() and gets his NFT.
- **Step 4**: I can make up infinite number of steps like that, actually the NICE person can keep on calling collectPresent() and collect his NFTs as many times as he wants, because the check that was supposed to prevent NICE people from collecting an NFT more than once checks the santaToken's balance that is only minted by EXTRA-NICE people. Hence this check is irrelevant for NICE people. For a matter of fact, according to the documentation and comments, it is also irrelevant for EXTRA-NICE people since they can buy a present for someone else by burning their santaTokens, thus finding a way to set their balance to 0 and be able to call collectPresent() again. However, buyPresent() is badly implemented.
- **Outcome**: NICE people can mint as many NFTs as they want.
- **Implications**: This vulnerability makes it possible for everyone NICE to collect an unlimited amount of NFTs just by calling a function again and again. The whole protocol is broken.

## Recommendation

Make the following changes:
Declare a mapping in the storage: ```mapping(address person => bool collected) private s_theListCheckedCollection;```
Remove this condition: ```if (balanceOf(msg.sender) > 0)```
Add this one instead:  ```if (s_theListCheckedCollection[msg.sender])```
Keep the revert.
Before the 2 return, set ```s_theListCheckedCollection[msg.sender] = true;```
