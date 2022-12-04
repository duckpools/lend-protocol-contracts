``scala 
{
	val minTxFee      = 1000000L
	val minBoxValue   = 1000000L
	
	val user          = SELF.R4[Coll[Byte]].get
	val requestAmount = SELF.R5[Long].get
	val publicRefund  = SELF.R6[Long].get
	
	val successor = OUTPUTS(1)
	
	val validSuccessorScript = successor.propositionBytes == user
	
	val validValue = successor.value >= requestAmount

	val multiBoxSpendSafety = successor.R7[Coll[Byte]].get == SELF.id
	
	val exchange = (
		validSuccessorScript &&
		validValue &&
		multiBoxSpendSafety && true && true
	)
	
	val deltaErg = SELF.value - successor.value
	
	val validDeltaErg = deltaErg <= minTxFee + minBoxValue
	val validHeight   = HEIGHT >= publicRefund
	
	val validTokens = if(successor.tokens.size != 0) {successor.tokens(0)._1 == SELF.tokens(0)._1 && successor.tokens(0)._2 == SELF.tokens(0)._2} else {false}
	
	val refund = (
		validSuccessorScript &&
		validDeltaErg &&
		multiBoxSpendSafety &&
		validHeight &&
		validTokens
	)
	
	sigmaProp(exchange || refund) 
}
```
