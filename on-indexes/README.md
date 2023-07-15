---
layout:
  title:
    visible: true
  description:
    visible: true
  tableOfContents:
    visible: true
  outline:
    visible: true
  pagination:
    visible: true
---

# On Indexes

Now that we understand how interest rates are modelled, let's look at how on-chain interest accrual works. Central to this is the concept of indexes.

Aave uses indexes to track interest and manage accruing interest on deposits and loans over time. This is crucial in the context of a blockchain, where events are tracked not by timestamps, but by blocks.

There are two types of indexes in Aave:

1. **Liquidity Index**: tracks supply interest accrued from inception to date
2. **Variable Borrow Index**: tracks variable borrow interest from inception to date

Both indexes represent the cumulative growth of interest over time for a specific asset, and continually increment over time.

{% hint style="info" %}
An index cannot decrement as cumulative interest always increases. Meaning to say, interest always adds up over time, however the actual interest rates themselves may vary.&#x20;
{% endhint %}

{% hint style="warning" %}
This positive relationship between indexes and time can be broken if we somehow introduced negative interest rates into Defi.&#x20;

This has been talked about in mainstream economics, with central banks experimenting with negative rates to stimulate growth. Hopefully, it will not come to pass.&#x20;
{% endhint %}

## Why use an index?

In traditional finance, interest is typically calculated based on the amount of time your money has been deposited. The level of granularity in such calculations can be such that valuations are updated on a per second basis, like in the banking system.&#x20;

However, blockchains operate based on block confirmations, not real-world time. Updating rates in real-time, on a per second basis would be extremely gas-intensive and not scalable approach.&#x20;

Hence, the solution was to use indexes, which would accumulate interest across blocks and updating the protocol on an ad-hoc basis.&#x20;

{% hint style="info" %}
Whenever a state-changing function is called: supply, withdraw, borrow, repay, etc, both indexes and rates are are updated.&#x20;
{% endhint %}

## How does it work?&#x20;

Instead of updating the index every block, Aave timestamps specific values of the index during state-changing transactions.

An index starts with a value of 1.0 at inception. As time passes and interest accrues, the liquidity index increases. For example, after a certain period, the liquidity index might be 1.05, indicating that 5% interest has been accrued on all deposits specific to that asset.

{% hint style="info" %}
This interest comes from borrowers who are paying interest on their loans.
{% endhint %}

### Index calculation

To calculate a new index value, the old index value is multiplied by the interest rate and the elapsed time.&#x20;

<figure><img src="../.gitbook/assets/image (147).png" alt="" width="420"><figcaption><p>Index formula</p></figcaption></figure>

#### **Assume**&#x20;

* ETH has just been made available on Aave for lending and borrowing - liquidity index starts at  **1.0**.&#x20;
* We have a user who initially deposits 10 ETH into the Aave lending pool.&#x20;

Now, let's consider the following changes in interest rates and their respective time periods:

| Month   | Interest Rate, annual |
| ------- | --------------------- |
| Month 1 | 12%                   |
| Month 2 | 6%                    |
| Month 3 | 8%                    |

To calculate the liquidity index at each month, we multiply the previous index by the factor `(1 + interest rate * time elapsed)`. Here's how the liquidity index evolves:

| Month          | Liquidity Index                       |
| -------------- | ------------------------------------- |
| End of Month 1 | 1 \* (1 + 0.12 \* 1/12) = 1.01        |
| End of Month 2 | 1.01 \* (1 + 0.06 \* 1/12) = 1.0105   |
| End of Month 3 | 1.0105 \* (1 + 0.08 \* 1/12) = 1.0113 |

Based on the evolving liquidity index, we can calculate how the user's deposit grows over time. Starting with 10 ETH, here's the growth of the user's deposit:

| Month          | Deposit                                  |
| -------------- | ---------------------------------------- |
| End of Month 1 | 10 ETH \* 1.01 = 10.1 ETH                |
| End of Month 2 | 10.1 ETH \* 1.0105 = 10.20105 ETH        |
| End of Month 3 | 10.20105 ETH \* 1.0113 = 10.31344665 ETH |

As the interest rates change over time, the liquidity index updates, and the user's deposit grows accordingly. The deposit value increases each month based on the compounded interest earned from the changing interest rates and the corresponding time periods.

### One Index to rule them all&#x20;

To accurately value each user's deposit and loan positions within the system, we will need to track the timestamps of all the instances of:

* deposits, withdrawals,&#x20;
* borrows taken, borrows repaid (in full or otherwise)

Then the headache of tracking multiple instances of these actions for just one user. In a tradfi setting, with a centralized database, one could simply spin up a system that tracks every action and position independently, and aggregate them all to value a user holistically.&#x20;

Tracking each action as an independent object in an on-chain system would be taxing to say the least. Hence, instead of initializing a liquidity index to track interest for each deposit - Aave has a single global liquidity index for each asset.&#x20;

Users' deposits and withdrawals are scaled against this global index. Let me illustrate.

#### Scaling Positions

Deposits in Aave are scaled down against the current liquidity index, at the time of deposit. Like so:

> amountScaled = Deposit Amount / Current Liquidity Index

In the same vein, existing balances are scaled up with liquidity index:

> Balance = amountScaled \* Current Liquidity Index

Since the liquidity index serves to reflect the interest accumulated since inception, deposits are divided against the liquidity index at time of deposit to negate all prior interest.

<figure><img src="../.gitbook/assets/image (8).png" alt=""><figcaption></figcaption></figure>

* User opts to deposit at t1, when Index = 1.1
* His deposit is scaled down by dividing against the index, to negate all interest prior to t1
* Should he choose to withdraw at t2, his scaled balance is scaled up by the index then, which accumulates **all** interest since inception, t0.

{% hint style="info" %}
We will talk about `_userState[account].balance` when we dive deeper into ATokens and how exactly user balances are stored and retrieved.&#x20;
{% endhint %}

{% hint style="success" %}
* Deposit interest is distributed across all depositors proportional to their balance and time committed.&#x20;
* A single index is used to track the accumulated interest for everyone.&#x20;
* The index and rate are updated for every deposit, borrow, withdrawal, repay of ANY Aave user.&#x20;
* The update works like this: Utilization Rate -> Variable Borrow rate -> Liquidity rate -> Liquidity Index
{% endhint %}

\