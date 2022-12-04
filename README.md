# Duckpools - A collateralized lending platform

## Introduction
Duckpools is a lending platform built on ergo that facilitates collateralized loans through algorithmic lending pools. It is the first dApp of its kind to be developed on Ergo and is the first lending platform that recognises ERG and Ergo assets as loan collateral. Duckpools features pooled lending which offers its lenders yield based on algorithmic, non-custodial interest rates defined by the protocol. 

## Architecture Overview
The duckpools protocol consists of a number of lending pools. Each pool holds a single Ergo native asset and exists as a single UTXO stored on-chain.

Each lend pool's UTXO allows for 5 different spending actions (Lend, Withdraw, Borrow, Repayment, Liquidation)


![Loan Diagram](https://i.ibb.co/b74fJcQ/pool-Loans.png)

### Lend Spending Path
The lend spending path allows lenders to add funds to a lending pool. For a lender to lend to the lending pool, they should simply recreate the lend pool UTXO with an increased amount of the asset to be lent. Similar to Ergodex's liquidity pool tokens, the lend pool UTXO releases a lending receipt token to lenders to recognise their share of funds in the pool. The amount of lending receipt tokens to be relased to a lender is governed by the following formula: 

**Lend Token Release Formula**
$$\frac{HeldAssets_{i}}{CirculatingLendTokens_{i}} <= \frac{HeldAssets_{f}}{CirculatingLendTokens_{f}}$$

Or in otherwords, a lender can withdraw any amount of lend tokens (i.e. increase the circulating supply of lend tokens) such that the value of the lend token (which logically is defined as $\frac{HeldAssets}{CirculatingLendTokens}$) does not decrease. It follows that the more a lender increases the Held Assets in the pool, the more lending receipt tokens they can obtain. 

### Withdraw Spending Path
The withdraw spending path allows lenders to redeem their lending pool tokens for their helds assets in the lending pool. Interestingly, the withdraw and lend spending paths can be combined since they are both governed by the lend token release formula. 

More explicitly, we see that when a lender redeems (i.e. reduce the circulating supply of lend tokens) the formula allows the lender to obtain an amount of held assets, provided that there remains enough held assets in the pool such that the lend token value does not decrease. 

We find two important results of the lend release formula:
1. Lenders with active loan positions cannot lose any amount of their held assets since no matter the spending action the lend token value does not decrease.
2. The value of the lend token can only increase over time, namely, when loans are repaid or liquidated with interest, the held assets will increase but the circulating supply of lend tokens will remain constant. In this way, lenders earn yield when borrowed funds are repaid / liquidated. This does have the drawback that lenders can only realise gains when loans are finalised. Continuous delivery of yield is left as an area of research for a future version of the protocol. 

### Borrow Spending Path 
The borrow spending path allows funds to be borrowed from the lending pool. The only check that the lending pool makes to allow borrowing is that some collateral is sent to the collateral address, where this collateral has a value over the minimum collateralization value (with this value calculated using the ErgoDex price). Repayment and liquidations conditions are defined by the collateral contract. Lending pools can accept any asset as collateral (provided the relevant trading pair exists on Ergodex), however, these assets and their minimum collaterialization percentages must be defined before the pool is bootstrapped.

Also note, to maintain a reference of the total assets held in a lending pool, a record of all assets utilised in loans is recorded in R4. In this way, the total held assets (which is used for calculating lend token value) is simply the sum of assets held in the pool UTXO and the assets on loan. The total assets on loan value is incremented whenever a borrower borrows from the pool (and decreases upon repayment/ liquidation). 

### Repayment / Liquidation Spending Path 
The repayment and liqudation rules are defined by the collateral contract, where the collateral can be unlocked if a repayment or liquidation is made. Repayments / liquidated assets are ultimately sent to the repayment contract which allows the asset to be sent to the lending pool. From the perspective of the lending pool, the repayment / liqudation spending path can be simply considered a "deposit" spending path, this is because the lending pool does not consider where assets are sourced; any assets can be accepted as a deposit to the pool. 

A deposit to the lending pool is defined as when assets are added to the pool and no value or lend tokens are withdrawn from the pool. The lend pool not only allows for deposits to be made but it also allows the utilised loans value in R4 of the pool UTXO to reduce by a maximum of the deposit value. Allowing the utilised loan value to reduce is important so that the lend token value can accurately reflect the sum of utilised and unutilised lent assets, as well as allowing the interest rate contract to obtain accurate utilisation values. 

On the surface, it may be seem dangerous to openly allow for any party to deposit assets to the pool and reduce the utilised assets value since by doing so the utilised assets value will become inaccurate. However, we cannot find an attack which would benefit any attacker in this system since all active loans ultimately require a repayment / liquidation by the rules of the collateral contract (so active borrowers don't gain) and on the flipside lenders don't lose since all active loans will eventually be repaid in full. As such, an attack of this nature would simply be gifting the pool assets for the sake of causing minimal relative inaccuracy to pool stats (interest rate). That said, duckpools is open to suggestions regarding this feature of the arctiecture. 

### Interest Rate Contract
Every lend pool has an interest rate UTXO which is a box governed by the interest rate contract that stores an array of historical interest rates calculated using the interest rate formula. Paramaters can be changed for the interest rate contract, however for the current ERG pool interest rates are calculated every 360 blocks using this formula:

**Annual Interest Rate Formula**
$$Rate = 0.005 + poolUtilPercentage * 0.13$$

Regardless of the formula used, it is important for interest rates to be dynamic based on pool utilization. The more a lend pool's assets are lent out, the higher the interest rate needs to be to accomodate for a decreasing supply of assets to be lent, otherwise, pool utilization can reach 100% and lenders would not be able to withdraw their assets until a loan is repaid or liquidated. On the flipside, when utilization is low the interest rate should lower to attract more borrowers to the pool and increase lender yield. 

The current structure of the interest rate UTXO is to store all interest rates from the start of a pool's history in an array. The interest accrued on a loan is then simply the sum of the interest rate entries in the interest rate UTXO since the loan's creation index (which is recorded in the collateral box) mutiplied by the loan value. This means Duckpools currently uses a simple interest rate model, however an adjustment to a compounding model is plausible for the future.

One thing to be considerate of is that using a single UTXO to record interest rates from the start of a pool's history will eventually succumb to the maximum UTXO size limit. With a 480 block update frequency, this limit may take over a year to reach, nonetheless a multi-UTXO interest rate system will be implemented before full launch to ensure this will not cause any problems. 


### Collateral Contract
The collateral contract dictates the conditions of the loan. The collateral contract is responsible for calculating the total amount owed on a loan (the sum of the loan value and interest accrued) and allows a repayment or liquidation spending path to unlock the held collateral. 

It is important for the collateral value to always remain above the loan value, as such, off-chain bots should regularly scan the collateral contract address and check if active loans can be liquidated. An active loan can be liquidated when its loan to collateral ratio drops below a liquidation threshold. For the ERG pool this threshold is set at 120%. Liquidation thresholds should generally be more agressive for volatile assets such that lenders funds are secured. 

Liquidation is executed through ErgoDex and the collateral contract ensures that the collateral is traded to the lend pool asset and is sent to the deposit contract. Currently there is no support for partial repayments, so loans must be repaid in full, likewise, the destination address of the repayment is the deposit contract.
















