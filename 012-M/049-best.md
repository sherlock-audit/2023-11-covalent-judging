Ripe Cloud Chameleon

high

# Incorrect Handling of ExchangeRate

## Summary

The `OperationalStaking.sol` contract experiences inconsistencies in calculating the `exchangeRate`, particularly when rewards are involved. The issue arises from the logic implemented after reward emission, where the `exchangeRate` fails to update accurately. This discrepancy leads to various miscalculations throughout the contract, miscalculations at `_sharesToTokens` and `_tokensToShares` opening window for manipulations.

## Vulnerability Detail

The staking uses shares and tokens mechanism and has an `exchangeRate` parameter which is one of the parameters of the `Validator` struct, it determines the rate of tokens and shares that the validator has, and it is being updated only when validators are being rewarded: 

```solidity
            v.exchangeRate += uint128((uint256(amount - commissionPaid) * uint256(DIVIDER)) / v.totalShares); 
```
Following this update, there are no subsequent points where the exchange rate is refreshed, this means that after validators got rewarded, the amount of tokens and shares will not be inline with the exchange rate, e.g. the `exchangeRate` can be `25 : 20` but actually 40 tokens with 36 shares, so `40 : 36` which is not equal to `25 : 20` ( `1.11 != 1.25` ), which can lead to miscalculations and manipulations by validators and delegators, and gaining more tokens then intended, posing a risk to the protocol's assets.

Let's illustrate the issue with an example, let's suppose that the actual numbers/values are greater than in the example and they are passing all the limits: `REWARD_REDEEM_THRESHOLD`, `DEFAULT_VALIDATOR_ENABLE_MIN_STAKE` and `DEFAULT_DELEGATOR_MIN_STAKE`, but for the simplicity in the example let's divide them to e.g. 100 and take the initial picture like validator has 20 tokens and as the initial exchange rate is `1 : 1`, the total shares are 20 as well ( `20 : 20` ).

Now suppose the validator collected 5 tokens of reward ( net rewards, without `commissionPaid` ):

```solidity
v.exchangeRate += uint128((uint256(amount - commissionPaid) * uint256(DIVIDER)) / v.totalShares);
// 10^18 + 5 * 10^18 / 20 = 25 / 20 * 10^18
```
The exchange rate is `25/20` now ( not writing 10^18 here, as it is not playing a role here ).  But the `stakings` are not updated. So subsequent actions, such as `stake`, `unstake`, `recoverUnstaking`, `redeemRewards` etc will change the `stakings` but the exchange rate will remain the same. For instance, if a validator or delegator stakes an additional 20 tokens via function `_stake` it will calculate how many shares it will add to a user:

```solidity
        uint128 sharesAdd = _tokensToShares(amount, v.exchangeRate);
```

And the `_tokensToShares` function looks like:

```solidity
    function _tokensToShares(uint128 amount, uint128 rate) internal pure returns (uint128) {
        return uint128((uint256(amount) * DIVIDER) / uint256(rate));
    }
// 20 * 10^18 / ( 21/20 * 10^18) = 16
```
So for the 20 tokens user gets 16 shares, now their token to share ratio is `40 : 36` while the exchange rate remains `25 : 20`.

Next, let's take the example that the user is unstaking all the amount ( 40 tokens ) i.e calling the `_unstake` function with `amount` parameter set to `0`.

The function calculates how much shares to remove:

```solidity
    uint128 sharesToRemove = _tokensToShares(effectiveAmount, v.exchangeRate); 

    function _tokensToShares(uint128 amount, uint128 rate) internal pure returns (uint128) {
        return uint128((uint256(amount) * DIVIDER) / uint256(rate));
    }
// 40 * 10^18 / (25/20 * 10^18) = 32
```

Now the user's token to share ratio is `0 : 4` `40 : 36 - 40 : 32` and they are calling `_redeemRewards` function with `amount` parameter set to `0` i.e it's requested to redeem all rewards.

The function is calculating:

```solidity
        // how many tokens a delegator/validator has in total on the contract
        // include earned commission if the delegator is the validator
        uint128 totalValue = _sharesToTokens(s.shares, v.exchangeRate);
```

here, the user has no tokens but 4 shares. The function calculates the token amount to be given to the user without checking whether the user has that token amount or not, despite the comment indicating: `how many tokens a delegator/validator has in total on the contract`.  So for 4 shares, it calculates:

```solidity
    function _sharesToTokens(uint128 sharesN, uint128 rate) internal pure returns (uint128) {
        return uint128((uint256(sharesN) * uint256(rate)) / DIVIDER);
    }
 // 4 * 25/20 * 10^18 /  10^18 = 5
```

5 tokens and then burning that 4 shares:

```solidity
        uint128 sharesToBurn = _tokensToShares(effectiveAmount, v.exchangeRate); 
```
This means the user received `45` tokens. The same result occurs when the user directly calls the `_redeemRewards` function with `amount` parameter set to `0` assuming token to share ratio `40 : 36`:

```solidity
        uint128 totalValue = _sharesToTokens(s.shares, v.exchangeRate);
// 36 * 25/20 * 10^18 /  10^18 = 45
```

## Impact

The lack of updating the `exchangeRate` parameter after a validator is rewarded introduces a vulnerability in the staking mechanism. This oversight can lead to discrepancies between the calculated exchange rate and the actual distribution of shares and tokens, allowing for potential manipulations by both validators and delegators. The vulnerability poses a security risk to the protocol by allowing malicious actors to exploit discrepancies in the exchange rate, leading to unintended economic consequences and possible financial losses for users and the protocol itself.

## Code Snippet

Here are the complete functions presented in the same order as in the **Vulnerability Detail** section (the relevant parts of the functions are shown in the **Vulnerability Detail** section)

https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L262-L319

https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L408-L446

https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L393-L395

https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L466-L537

https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L589-L635

https://github.com/sherlock-audit/2023-11-covalent/blob/main/cqt-staking/contracts/OperationalStaking.sol#L386-L388

## Tool used

Manual Review

## Recommendation

Develop a mechanism that ensures the continuous and dynamic calculation of the exchange rate based on the actual token-to-share ratios. This mechanism should be triggered on every change in both token and share balances.
Consider adopting established standards for token staking mechanisms, such as **ERC4626**, which provides guidelines for consistent and secure exchange rate calculations.