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
- **Implications**: No more liquidity

## Recommendation
The Upgraded is a bit better than the V1 as the exchange rate gets higher but still room for reentrancy so...
Make the following change:

```diff
- assetToken.mint(msg.sender, mintAmount);
+ token.safeTransferFrom(msg.sender, address(assetToken), amount);

- token.safeTransferFrom(msg.sender, address(assetToken), amount);
+ assetToken.mint(msg.sender, mintAmount);

```
## Proof of Concept for [Reentrancy Attack 3]

### Overview:
Attacker can do a reentrancy attack using the above function and taking several loans and repaying it as 1.

### Actors:
- **Attacker**: the malicious user.
- **Victim**: other users, LPs, ThunderLoan.
- **Protocol**: The ThunderLoan contract itself.

### Exploit Scenario:
- **Initial State**: The protocol is deployed
- **Step 1**: the attacker decides to take a flash Loan using a malicious contract that calls the flashLoan() function.
- **Step 2**: the protocol makes the error of making an external call to the attacker's malicious contract that contains a receive() that will call back flashLoan() with the same amount, that will decrease the startingBalance.
- **Step 3**: The attacker gets away by repaying what he's owed for his last flashLoan request.
- **Outcome**: Attacker steals a lot of fund providing he repays for his last request.
- **Implications**: Victims will lose their funds

## Recommendation

Make the following change:

```diff
- if (!receiverAddress.isContract()) {
-     revert ThunderLoan__CallerIsNotContract();
- }
+ if (receiverAddress.isContract()) {
+     revert ThunderLoan__CallerIsNotContract();
+ }

```
## Proof of Concept for [Bad Logic Implementation]

### Overview:
There's a bad logic implementation in the redeem(IERC20 token, uint256 amountOfAssetToken) function, that vulnerability the LP to redeem more than what he's owed.

### Actors:
- **Attacker**: the malicious LP that wants to redeem his funds.
- **Victim**: Other LPs.
- **Protocol**: The ThunderLoan contract itself.

### Exploit Scenario:
- **Initial State**: Protocl is depolyed, Attacker has already deposited some amounts.
- **Step 1**: We know amountUnderlying depands on amountOfAssetToken because:
uint256 amountUnderlying = (amountOfAssetToken * exchangeRate) / assetToken.EXCHANGE_RATE_PRECISION();
LPs can redeem more than they are owed from the protocol if they input a certain amountOfAssetToken such that:
amountOfAssetToken > assetToken.balanceOf(msg.sender) && amountOfAssetToken < type(uint256).max.
In order to max their profits they can input: amountOfAssetToken = type(uint256).max - 1.
The attacker will also burn more than he should: assetToken.burn(msg.sender, amountOfAssetToken);
- **Outcome**: The attacker can redeem much more than what he's owed but will also burn more than he should

## Recommendation

Make the following change:

```diff
- if (amountOfAssetToken == type(uint256).max) {
-     amountOfAssetToken = assetToken.balanceOf(msg.sender);
- }
+ require (amountOfAssetToken >= assetToken.balanceOf(msg.sender), "Not enough assetToekn in msg.sender's balance")

```
