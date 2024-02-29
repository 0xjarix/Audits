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

## H1 [safeMint reentrancy]

### Overview:
CEI pattern was not respected in the mintRapper() function, alowing attacker to mint several tokens with better default attributes.

### Actors:
- **Attacker**: the malicious minter.
- **Protocol**: The SantasList contract itself.

### Exploit Scenario:
- **Initial State**: The Protocol is already deployed and the people are calling the mintRapper() function.
- **Step 1**: The Attacker creates a malicious contract calls mintRapper and performs a reentrant call inside the onERC721Received callback that he would also have implemented to allow his contract receiving the NFTs.
- **Outcome**: Attacker already has attributes worth 3 days of staking, without the credTokens of course
  RapperStats({weakKnees: false, heavyArms: false, spaghettiSweater: false, calmAndReady: false, battlesWon: 0});
- **Implications**: Attacker has street experience without getting to the street, he can mint several NFTs and make them participate in rap battles

## Recommendation

Make the following changes:
```
function mintRapper() public {
        uint256 tokenId = _nextTokenId++;

        // Initialize metadata for the minted token
        rapperStats[tokenId] =
            RapperStats({weakKnees: true, heavyArms: true, spaghettiSweater: true, calmAndReady: false, battlesWon: 0});
        _safeMint(msg.sender, tokenId);
    }
```

## H2 [Weak PRNG]
## Overview
weak PRNG in _battle() that makes it possible for the defender to win any battle.

## Vulnerability Details
in this line:
uint256 random =
            uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao, msg.sender))) % totalBattleSkill;
The defender already knows msg.sender since it is the address of the contract he's interacting with.
The defenfer already can decide the block.timestamp, since he's the one calling goOnStageOrBattle(), he can decide when to call it exactly.
The defender already knows totalBattleSkill since it's the sum of his skill and his opponent's
block.prevandao reads the RANDAO mix generated in the previous block. (prev block.difficulty)
`random` should be unknown to the sender, then the block.prevrandao is useless.

## Impact
Defender can win any battle and gets defenderBet and _credBet

## Tools Used
Slither

## Recommendations
Use chainlink oracle for randomness
