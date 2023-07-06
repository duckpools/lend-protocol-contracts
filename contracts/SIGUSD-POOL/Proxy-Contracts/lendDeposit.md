```scala
{
	val lendPoolToken = fromBase58("13Sg6DCdibigKGQnYzBxZtyXRt8W5K5wqY9LTgSzmEhH")
	val minTxFee      = 1000000L
	val minBoxValue   = 1000000L
	
	val user          = SELF.R4[Coll[Byte]].get
	val requestAmount = SELF.R5[Long].get
	val publicRefund  = SELF.R6[Long].get
	
	
	if (OUTPUTS.size < 3) {
		val refundBox = OUTPUTS(0)
		val deltaErg = SELF.value - refundBox.value
		val validDeltaErg = deltaErg <= minTxFee
		val validRefundScript = refundBox.propositionBytes == user
		val validHeight = HEIGHT >= publicRefund
		val multiBoxRefund = if (refundBox.R4[Coll[Byte]].isDefined) refundBox.R4[Coll[Byte]].get == SELF.id else false
		val refund = (
			validRefundScript &&
			validDeltaErg &&
			multiBoxRefund &&
			validHeight
		)
		refund
	} else {
		val successor = OUTPUTS(2)
		val exchangedTokens = successor.tokens
		
		val validSuccessorScript = successor.propositionBytes == user
		val validTokens = if (exchangedTokens.size > 0){exchangedTokens(0)._1 == lendPoolToken && exchangedTokens(0)._2 >= requestAmount} else {false}
		val multiBoxSpendSafety = if (successor.R7[Coll[Byte]].isDefined) successor.R7[Coll[Byte]].get == SELF.id else false
		
		val exchange = (
			validSuccessorScript &&
			validTokens &&
			multiBoxSpendSafety
		)
		exchange
	}
}
