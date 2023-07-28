```scala
{    
	if (OUTPUTS.size >= 3) {
		
		val neededAmount  = SELF.R4[Long].get
		val borrower      = SELF.R5[Coll[Byte]].get
		
		val borrowerBox = OUTPUTS(0)
		
		val validBorrowerScript = borrowerBox.propositionBytes == borrower
		val validBorrowerCollateral = borrowerBox.value >= neededAmount
		
		val multiBoxSpendSafety = borrowerBox.R4[Coll[Byte]].get == SELF.id
		sigmaProp(
			validBorrowerScript &&
			validBorrowerCollateral &&
			multiBoxSpendSafety &&
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
