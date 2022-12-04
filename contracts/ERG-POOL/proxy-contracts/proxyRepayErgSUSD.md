```scala
{    
	if (OUTPUTS.size >= 3) {
		val sigUSDTokenId = fromBase58("GYATox71P9XAERmzoDdTGELa62f5ALyjxJLRSfJfKsh")
		
		val neededAmount  = SELF.R4[Long].get
		val borrower      = SELF.R5[Coll[Byte]].get
		
		val borrowerBox = OUTPUTS(0)
		
		val validBorrowerScript = borrowerBox.propositionBytes == borrower
		val validBorrowerTokens = borrowerBox.tokens(0)._1 == sigUSDTokenId && borrowerBox.tokens(0)._2 >= neededAmount
		
		val multiBoxSpendSafety = borrowerBox.R4[Coll[Byte]].get == SELF.id
		sigmaProp(
			validBorrowerScript &&
			validBorrowerTokens &&
			multiBoxSpendSafety 
		)
	} else {
		val minTxFee = 1000000L
		val borrower = SELF.R5[Coll[Byte]].get
		
		val refundBox = OUTPUTS(0)
		
		val validBorrowerScript = refundBox.propositionBytes == borrower
		val validRefundValue = refundBox.value >= SELF.value - minTxFee
		sigmaProp(
			validBorrowerScript &&
			validRefundValue &&
			HEIGHT >= SELF.R6[Int].get
		)
	}	
}
```
