# Voting Booth security review by 0xjarix

*********************review commit hash -********************* **[5b9554656d53baa2086ab7c74faf8bdeaf81a8b7](https://github.com/Cyfrin/2023-12-Voting-Booth)**

### The code was reviewed for a total of 2 hours.
---

# High Severity Finding
## H0 - Proof of Concept for [Bad logic imlementation]

### Overview:
VotersFor rewards is less than it should be and funds are locked forever due to bad logic implementation.

### Actors:
- **Victims**: VotersFor when there are votersAgainst
- **Protocol**: the VotingBooth contract

### Exploit Scenario:
- **Initial State**: The Protocol is deployed by its creator who sent 1 ETH as minimum funding and gave as argument a list of 5 voters. Hence when the 3rd voter votes, the voting will be considered complete.
- **Step 1**: Victim#1 who is voters[0] in the PoC below votes for the proposal by calling booth.vote(true)
- **Step 2**: voters[2] in the PoC below votes against the proposal by calling booth.vote(false)
- **Step 3**: Victim#2 who is voters[2] in the PoC below votes for the proposal by calling booth.vote(true)
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.23;

import {VotingBooth} from "../src/VotingBooth.sol";
import { Script } from "forge-std/Script.sol";
import { console } from "forge-std/console.sol";

contract Poc is Script {
    function run() public {
        address[] memory voters = new address[](5);
        voters[0]     = address(0x1);
        voters[1]     = address(0x2);
        voters[2]     = address(0x3);
        voters[3]     = address(0x4);
        voters[4]     = address(0x5);
        address creator = address(0x6);
        vm.deal(creator, 1e18);
        vm.startPrank(creator);
        VotingBooth booth = new VotingBooth{value: 1e18}(voters);
        vm.stopPrank();
        vm.startPrank(voters[0]);
        booth.vote(true);
        vm.stopPrank();
        vm.startPrank(voters[1]);
        booth.vote(false);
        vm.stopPrank();
        vm.startPrank(voters[2]);
        booth.vote(true);
        vm.stopPrank();
        if (voters[0].balance != 5e17){
            console.log("voters[0].balance: %d", voters[0].balance);
            console.log("This should be 5e17");
        }
        if (voters[2].balance != 5e17){
            console.log("voters[2].balance: %d", voters[2].balance);
            console.log("Same for this one");
        }
        if (address(booth).balance != 0) {
            console.log("booth.balance: %d", address(booth).balance);
            console.log("funds are locked here forever!");
        }
        console.log("creator.balance: %d", creator.balance);
    }
}
```
- **Outcome**: victim#1 gets rewarded 0.33...33 ETH instead of 0.5 ETH and victim#2 gets rewarded 0.33...34 ETH instead of 0.5 ETH. The funds that have not been distributed will remain locked inside the protocol.
- **Implications**: Funds will be locked inside the protocol and the voters who voted for the proposol will receive less than they deserve. In the PoC above I assumed the creator sends 1 ether which is the minimum funding, had he send more, the amount of funds locked would increase as well. Voters can predict how much they're owed as nothing is hashed and everything is available onchain and people can read the storage and know who voted for and who voted against the proposal. That would leave them angry at the protocol and the creator.
  
## Recommendation

Make the following change:

```diff
- uint256 rewardPerVoter = totalRewards / totalVotes;
+ uint256 rewardPerVoter = totalRewards / totalVotesFor;
```
