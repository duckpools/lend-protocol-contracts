```scala
{
	// Constants
	val transactionFee = 1000000L 
	val MaxBorrowTokens = 9000000000000000L // Maximum allowed borrowable tokens
	val PoolNft = fromBase58("8txwdGRnbi6RfPhR7rxgzvk5yaxzmm4PGXTaWjfVsfJa") // Non-fungible token for the pool
	
	val initalPool = INPUTS(0)
	val finalPool = OUTPUTS(0)
	
	// Amount borrowed
	val loanAmount = SELF.tokens(0)._2
	val repaymentTokens = SELF.tokens(1)
	val repaymentAmount = repaymentTokens._2
	
	// Find change in the pools borrow tokens
	val borrow0 = MaxBorrowTokens - initalPool.tokens(2)._2
	val borrow1 = MaxBorrowTokens - finalPool.tokens(2)._2	
	val deltaBorrowed = borrow0 - borrow1
	
	val assetsInPool0 = initalPool.tokens(3)._2
	val assetsInPool1 = finalPool.tokens(3)._2
	val deltaPoolAssets = assetsInPool1 - assetsInPool0
	
	// Check if the pool NFT matches in both initial and final state
	val validFinalPool = finalPool.tokens(0)._1 == PoolNft
	val validInitialPool = initalPool.tokens(0)._1 == PoolNft
	
	// Calculate the change in the value of the pool
	val deltaValue = finalPool.value - initalPool.value
	
	// Check if the delta value is greater than or equal to the loan amount minus the transaction fee
	val validValue = deltaValue >= SELF.value - transactionFee
	// Check if the delta between borrowed amounts is equal to the loan amount
	val validBorrowed = deltaBorrowed == loanAmount
	// Check repayment assets go to pull
	val validRepayment = repaymentAmount == deltaPoolAssets && repaymentTokens._1 == initalPool.tokens(3)._1
	
	// Check that SELF is INPUTS(1) to prevent same box attack
	val multiBoxSpendSafety = INPUTS(1) == SELF
	
	// Combine all the conditions into a single Sigma proposition
	sigmaProp(
		validFinalPool &&
		validInitialPool &&
		validValue &&
		validRepayment &&
		validBorrowed &&
		multiBoxSpendSafety
	)
}
```
