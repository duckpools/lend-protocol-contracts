```scala
{
	val minTxFee      = 1000000L
	val minBoxValue   = 1000000L
	
	val user          = SELF.R4[Coll[Byte]].get
	val requestAmount = SELF.R5[Long].get
	val publicRefund  = SELF.R6[Long].get
	val requestedToken = SELF.R7[Coll[Byte]].get
	
	if (OUTPUTS.size < 3) {
		val refundSuccessor = OUTPUTS(0)
		val deltaErg = SELF.value - refundSuccessor.value
		
		val validRefundSuccessor = refundSuccessor.propositionBytes == user
		val validDeltaErg = deltaErg <= minTxFee
		val validHeight   = HEIGHT >= publicRefund
		val multiBoxSafetyRefund =  if (refundSuccessor.R4[Coll[Byte]].isDefined) refundSuccessor.R4[Coll[Byte]].get == SELF.id else false
		val validTokens = if(refundSuccessor.tokens.size != 0) refundSuccessor.tokens(0) == SELF.tokens(0) else false
	
		
		// Refund conditions
		val refund = (
			validRefundSuccessor &&
			validDeltaErg &&
			multiBoxSafetyRefund &&
			validHeight &&
			validTokens && true && true && true
		)
		refund
	
	} else {
		// Withdrawal conditions
		val successor = OUTPUTS(2)
		
		val validSuccessorScript = successor.propositionBytes == user		
		val validTokens = successor.tokens(0)._2 >= requestAmount && successor.tokens(0)._1 == requestedToken
		val multiBoxSpendSafety = if (successor.R7[Coll[Byte]].isDefined) successor.R7[Coll[Byte]].get == SELF.id else false
		
		val exchange = (
			validSuccessorScript &&
			validTokens &&
			multiBoxSpendSafety 
		)
		exchange	
	}
}
```
