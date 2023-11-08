# ThunderLoan security review by 0xjarix

*********************review commit hash -********************* **[026da6e73fde0dd0a650d623d0411547e3188909](https://github.com/Cyfrin/2023-11-Thunder-Loan)**

### The code was reviewed for a total of 3 hours.
---


## Proof of Concept for [Reentrancy Attack 1]

### Overview:
Checks-Effects-Interactions order was not respected in the deposit(IERC20 token, uint256 amount) function allowing the attacker to call deposit(IERC20 token, uint256 amount) of ThunderLoan till he drains the contract's funds

### Actors:
- **Attacker**: the malicious LP who's going to perform the reentrancy attack.
- **Victim**: Other LPs.
- **Protocol**: The ThunderLoan contract itself.

### Exploit Scenario:
- **Initial State**: The ThunderLoan contract already contains liquidity thanks to other LPs.
- **Step 1**: Attacker deposit a certain amount of token.
- **Step 2**: Attacker calls the deposit(IERC20 token, uint256 amount) function with his malicious contract.
- **Step 3**: When deposit(IERC20 token, uint256 amount) is called, the protocol makes the precious error of making an external call to the attacker's contract before updating the exchange rate AND actually depositing the funds through token.safeTransferFrom(msg.sender, address(assetToken), amount);.
- **Step 4**: The attacker doesn't forget to add a receive() or fallback() function to his malicious contract that will keep on calling the  function till he drains the protocol's contract from all its funds.
- **Outcome**: The protocol will lose a lot of funds.
- **Implications**: No more liquidiy

## Recommendation

Make the following change:

```diff
- assetToken.mint(msg.sender, mintAmount);
+ uint256 calculatedFee = getCalculatedFee(token, amount);
+ assetToken.updateExchangeRate(calculatedFee);
+ token.safeTransferFrom(msg.sender, address(assetToken), amount);

- uint256 calculatedFee = getCalculatedFee(token, amount);
- assetToken.updateExchangeRate(calculatedFee);
- token.safeTransferFrom(msg.sender, address(assetToken), amount);
+ assetToken.mint(msg.sender, mintAmount);

```
## Proof of Concept for [Reentrancy Attack 2]

### Overview:
Checks-Effects-Interactions order was not respected in the deposit(IERC20 token, uint256 amount) function allowing the attacker to call deposit(IERC20 token, uint256 amount) of ThunderLoanUpgraded till he drains the contract's funds

### Actors:
- **Attacker**: the malicious LP who's going to perform the reentrancy attack.
- **Victim**: Other LPs.
- **Protocol**: The ThunderLoanUpgraded contract itself.

### Exploit Scenario:
- **Initial State**: The ThunderLoanUpgraded contract already contains liquidity thanks to other LPs.
- **Step 1**: Attacker deposit a certain amount of token.
- **Step 2**: Attacker calls the deposit(IERC20 token, uint256 amount) function with his malicious contract.
- **Step 3**: When deposit(IERC20 token, uint256 amount) is called, the protocol makes the precious error of making an external call to the attacker's contract before actually depositing the funds through token.safeTransferFrom(msg.sender, address(assetToken), amount);.
- **Step 4**: The attacker doesn't forget to add a receive() or fallback() function to his malicious contract that will keep on calling the  function till he drains the protocol's contract from all its funds.
- **Outcome**: The protocol will lose all its funds.
- **Implications**: The players that entered the same raffle as the attacker are victims of this exploit as players cannot get refunded anymore, the owner will not get his fees and the winner will not get his prize amount.

## Recommendation
The Upgraded is a bit better than the V1 as the exchange rate gets higher but still room for reentrancy so...
Make the following change:

```diff
- assetToken.mint(msg.sender, mintAmount);
+ token.safeTransferFrom(msg.sender, address(assetToken), amount);

- token.safeTransferFrom(msg.sender, address(assetToken), amount);
+ assetToken.mint(msg.sender, mintAmount);

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
- **Step 2**: ```require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");``` fails when the victim calls withdrawFees().
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
## Proof of Concept for [Logic Implementation]

### Overview:
There's a bad logic implementation in the getActivePlayerIndex() function, that vulnerability enables the first to enter the raffle to get refunded whenever he wants.

### Actors:
- **Attacker**: the malicious player EOA that is the first to enter the raffle, his index is 0.
- **Victim**: Any player that entered the same raffle as the attacker.
- **Protocol**: The raffle contract itself.

### Exploit Scenario:
- **Initial State**: Protocl is depolyed.
- **Step 1**: The attacker is the first to enter the raffle, his index is 0
- **Step 2**: The attacker creates a malicious contract to call the getActivePlayerIndex() by passing in argument the address of the malicious contract that will not be found.
- **Step 3**: A bad implementation of the getActivePlayerIndex() will return 0 when the address will not be found.
- **Step 4**: The malicious contract will impersonate the attacker's initial address to be able to get a refund 
- **Outcome**: The attacker can get refunded twice
- **Implications**: The attacker can get refunded a lot more if he creates other malicious contracts

## Recommendation

Make the following change:

```diff
+ function getActivePlayerIndex(address player) external view returns (int128) {
        for (uint256 i = 0; i < players.length; i++) {
            if (players[i] == player) {
                return i;
            }
        }
+       return -1;
    }
- function getActivePlayerIndex(address player) external view returns (uint256) {
        for (uint256 i = 0; i < players.length; i++) {
            if (players[i] == player) {
                return i;
            }
        }
-       return 0;
    }

```
