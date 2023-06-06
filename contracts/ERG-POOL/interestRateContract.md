```scala
{
	val PoolNft                = fromBase58("CemYwVWXuAbZu1qvJ313DQrN3tuvCmD3svcsqHfXJzMB")
	val ParamaterBoxNft = fromBase58("9M8VueZ5srjdUmi9T2sfkHCZfQPabFbr6EkEiBfP7LWn")
	val InterestDenomination     = 1000000000L
	val CoefficientDenomination = 100
	val InitiallyLockedLP      = 9000000000000000L
	val MaximumBorrowTokens        = 9000000000000000L
	val MaximumHistoryHeight = 5
	val MaximumExecutionFee = 2000000
	val updateFrequency = 7

	val successor = OUTPUTS(0)
	val pool      = CONTEXT.dataInputs(0)
	val parameterBox = CONTEXT.dataInputs(1)

	val interestHistory      = SELF.R4[Coll[Long]].get
	val recordedHeight       = SELF.R5[Long].get
	val childIndex = SELF.R6[Int].get
	val finalInterestHistory = successor.R4[Coll[Long]].get
	val finalHeight          = successor.R5[Long].get
	val finalChildIndex = successor.R6[Int].get

	// get coefficients
	val coefficients = parameterBox.R4[Coll[Long]].get
	val a = coefficients(0).toBigInt
	val b = coefficients(1).toBigInt
	val c = coefficients(2).toBigInt
	val d = coefficients(3).toBigInt
	val e = coefficients(4).toBigInt
	val f = coefficients(5).toBigInt
	 
	val deltaHeight      = HEIGHT - recordedHeight
	val isReadyToUpdate = deltaHeight >= updateFrequency // About 12 hours

	val deltaFinalHeight      = finalHeight - HEIGHT
	val validDeltaFinalHeight = (deltaFinalHeight >= 0 && deltaFinalHeight <= 30)
	val borrowed    = MaximumBorrowTokens - pool.tokens(2)._2
	val util        = (InterestDenomination.toBigInt * borrowed.toBigInt / (pool.value.toBigInt + borrowed.toBigInt)).toLong

	val D = CoefficientDenomination.toBigInt
	val M = InterestDenomination
	val x = util.toBigInt

	val currentRate = (
		M + (
			a + 
			(b * x) / D + 
			(c * x) / D * x / M +
			(d * x) / D * x / M * x / M +
			(e * x) / D * x / M * x / M * x / M + 
			(f * x) / D * x / M * x / M * x / M * x / M
			).toLong
		) 

	val retainedERG          = successor.value >= SELF.value - MaximumExecutionFee
	val preservedInterestNFT = successor.tokens == SELF.tokens

	val validSuccessorScript = SELF.propositionBytes == successor.propositionBytes
	val validInterestUpdate  = interestHistory.append(Coll(currentRate)) == finalInterestHistory

	val validPoolBox = pool.tokens(0)._1 == PoolNft
	val validParameterBox = parameterBox.tokens(0)._1 == ParamaterBoxNft

	val retainIndex = finalChildIndex == childIndex
	val isUnderMaxHeight = finalInterestHistory.size <= MaximumHistoryHeight

	sigmaProp(
		isReadyToUpdate &&
		isUnderMaxHeight &&
		validSuccessorScript &&
		retainedERG &&
		preservedInterestNFT &&
		validInterestUpdate &&
		validDeltaFinalHeight &&
		retainIndex &&
		validPoolBox &&
		validParameterBox 	  	  
	)
}
```
