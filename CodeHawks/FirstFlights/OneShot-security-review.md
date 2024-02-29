# One Shot security review by 0xjarix
---

## L0 [PUSH0 on arbitrum]

### Overview:
PUSH0 is not supported on Arbitrum for when we use versions of solidity >=0.8.20

## Recommendation
Change to solidity 0.8.19

## H0 [NFT not staked when stake() is called]

## Summary
When staked, the owner of the NFT is still the user, but it is supposed to be the street contract for all the duration of the staking.

## Vulnerability Details
In fact, the street contract never receive the NFT since onERC721Received is never triggered since transferFrom() doesn't call the checkOnERC721Received, unlike safeTransferFrom(). Also stakes[tokenId] is set before transferFrom is called, and since transferFrom will not revert, the call is successful


## Impact
User can do whatever he wants with his NFT when he's staking as it is not technically in the street contract

## Tools Used
Manual analysis

## Recommendations
Use safeTransferFrom instead of transferFrom in the stake() function and modify stake so it looks like this:
function stake(uint256 tokenId) external {
        oneShotContract.safeTransferFrom(msg.sender, address(this), tokenId);
        stakes[tokenId] = Stake(block.timestamp, msg.sender);
        emit Staked(msg.sender, tokenId, block.timestamp);
    }

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
