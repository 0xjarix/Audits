## Gas Optimization
The following function can be rewritten in a more optimized manner.
```solidity
function isHappyHorse(uint256 horseId) external view returns (bool) {
        if (horseIdToFedTimeStamp[horseId] <= block.timestamp - HORSE_HAPPY_IF_FED_WITHIN) {
            return false;
        }
        return true;
    }
```
More gas-efficient version:
```solidity
function isHappyHorse(uint256 horseId) external view returns (bool) {
        return (horseIdToFedTimeStamp[horseId] > block.timestamp - HORSE_HAPPY_IF_FED_WITHIN)
    }
```
We avoid usging the opcodes linked to the `if` statement and also those linked to the `<=` since in the EVM the LTE (Less Than Or Equal) opcode doesn't not exist so `GT` (Greater Than) and `ISZERO` are used. Now we are only using `GT`.
