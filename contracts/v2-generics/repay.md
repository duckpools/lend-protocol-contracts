```scala
{    
	if (OUTPUTS.size >= 3) {
		
		val neededAmount  = SELF.R4[Long].get
		val borrower      = SELF.R5[Coll[Byte]].get
		val tokenAmounts = SELF.R8[Coll[Long]].get
		val tokenIds = SELF.R9[Coll[Coll[Byte]]].get
		
		val borrowerBox = OUTPUTS(0)
		
		val validBorrowerScript = borrowerBox.propositionBytes == borrower
		val validBorrowerCollateral = borrowerBox.value >= neededAmount

		// Make sure borrowerbox tokens matches the tokenAmounts tokenIds
		val expectedTokens = tokenIds.zip(tokenAmounts)
        val validBorrowerTokens = borrowerBox.tokens == expectedTokens
		
		val multiBoxSpendSafety = borrowerBox.R4[Coll[Byte]].get == SELF.id
		sigmaProp(
			validBorrowerScript &&
			validBorrowerCollateral &&
			validBorrowerTokens &&
			multiBoxSpendSafety
		)
	} else {
		val minTxFee = 1000000L
		val borrower = SELF.R5[Coll[Byte]].get
		
		val refundBox = OUTPUTS(0)
		
		val validBorrowerScript = refundBox.propositionBytes == borrower
		val validRefundValue = refundBox.value >= SELF.value - minTxFee
		val validRefundTokens = refundBox.tokens == SELF.tokens
		sigmaProp(
			validBorrowerScript &&
			validRefundValue &&
			validRefundTokens &&
			HEIGHT >= SELF.R6[Int].get
		)
	}	
}
```
