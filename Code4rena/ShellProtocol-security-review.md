# Shell Protocol List security review by 0xjarix

*********************review code base -********************* **[Shell Protocol's code base](https://github.com/code-423n4/2023-11-shellprotocol/tree/main)**

### The code was reviewed for a total of 1 hour.
---


## H0 [Bad Math Arithmetic in convertDecimals()]

### Overview:
The convertDecimals() function inside the OceanAdapter contract performs bad arithmetic. This function is called twice inside each of the following contracts: Curve2PoolAdapter and CurveTriCryptoAdapter, in the primitiveOutputAmount() function of these 2 contracts.

### Actors:
- **Victim**: the user of the shell protocol
- **Attacker**: the pool that is calling this function
- **Protocol**: The Shell Protocol

### Exploit Scenario:
- **Initial State**: The Protocol is already deployed.
- **Step 1**: the Attacker calls the primitiveOutputAmount() with the ID of USDC as inputToken and the ID of WETH as outputToken, inputAmount equal to 99,999,999,999 and a minimumOutputAmount equal to 0
- **Step 2**: The primitiveOutputAmount() calls the _convertDecimals() a first time to set rawInputAmount
- **Step 4**: since USDC uses only 6 decimals and WETH uses 18, the _convertDecimals() will enter this part of the function:
  ```
  } else {
            // Decimal shift right (remove precision) -> truncation
            uint256 shift = 10 ** (uint256(decimalsFrom - decimalsTo));
            convertedAmount = amountToConvert / shift;
        }
  ```
- **Step 5**: shift = 10^12
- **Step 5**: convertedAmount = amountToConvert / shift = 99,999,999,999 / 10^12 = 0
- **Outcome**: rawInputAmount == 0
- **Impact**: The pool will be able to give an outputAmount corresponding to the rawInputAmount = 0
