```scala
{
	val emissionBlock = 1210027
	val precision = 1000000000L
		
	// Initial State
	val iScript = SELF.propositionBytes
	val iValue = SELF.value
	val iTokens = SELF.tokens
	val iReceiptTokens = SELF.tokens(0)
	val iQuacks = SELF.tokens(1)
	val iTotalQuacks = SELF.R4[Long].get
	val iAssetAmounts = SELF.R5[Coll[Long]].get
	val iErgAmount = SELF.R6[Long].get

	// Final State
	val successor = OUTPUTS(0)
	val fScript = successor.propositionBytes
	val fValue = successor.value
	val fTokens = successor.tokens
	val fReceiptTokens = successor.tokens(0)
	val fQuacks = successor.tokens(1)
	val fTotalQuacks = successor.R4[Long].get
	val fAssetAmounts = successor.R5[Coll[Long]].get
	val fErgAmount = successor.R6[Long].get
	
	val deltaReceiptTokens = fReceiptTokens._2 - iReceiptTokens._2
	val deltaQuacks = fQuacks._2 - iQuacks._2

	if (HEIGHT < emissionBlock) {
		// Allow deposits
		val isValidExchange = deltaQuacks == -1 * deltaReceiptTokens 
		val isDeltaQuacksPositive = deltaQuacks > 0
		
		val retainScript = iScript == fScript
		val retainValue = iValue == fValue
		val retainTokens = iTokens.slice(2, iTokens.size) == fTokens.slice(2, iTokens.size)
		val retainTokenSize = iTokens.size == fTokens.size
		val retainReceiptId = iReceiptTokens._1 == fReceiptTokens._1
		val retainQuacksId = iQuacks._1 == fQuacks._1 
		val retainAssetAmounts = fAssetAmounts == iAssetAmounts
		val retainErgAmounts = fErgAmount == iErgAmount
		val isValidTotalQuacks = fTotalQuacks == iTotalQuacks + deltaQuacks
		
		isValidExchange &&
		isDeltaQuacksPositive &&
		retainScript &&
		retainValue &&
		retainTokens &&
		retainTokenSize &&
		retainReceiptId &&
		retainQuacksId &&
		retainAssetAmounts &&
		retainErgAmounts &&
		isValidTotalQuacks
	} else {
		// Allow withdrawals
		val userShare = ((deltaReceiptTokens.toBigInt * precision.toBigInt).toBigInt / iTotalQuacks.toBigInt).toBigInt
		val isValidValue = fValue.toBigInt >= iValue.toBigInt - (userShare * iErgAmount.toBigInt) / precision.toBigInt
		val slicedExpectedTokens = iTokens.slice(1, iTokens.size)
		val zippedTokens = slicedExpectedTokens.zip(iAssetAmounts)
		val expectedTokens = zippedTokens.map{
				(tokenTuple : ((Coll[Byte], Long), Long)) => 
				(tokenTuple(0)._1, (tokenTuple(0)._2.toBigInt -  (userShare * tokenTuple(1).toBigInt) / precision.toBigInt))
		}
		
		val outputTokensBigInt = fTokens.slice(1, iTokens.size).map{
				(token : (Coll[Byte], Long)) => 
				(token._1, token._2.toBigInt)
		}
		val isValidTokens = expectedTokens == outputTokensBigInt
		
		val retainScript = iScript == fScript
		val retainTokenSize = iTokens.size == fTokens.size
		val retainReceiptId = iReceiptTokens._1 == fReceiptTokens._1
		val retainTotalQuacks = iTotalQuacks == fTotalQuacks	
		val retainAssetAmounts = fAssetAmounts == iAssetAmounts
		val retainErgAmounts = fErgAmount == iErgAmount
		
		isValidValue &&
		isValidTokens &&
		retainScript &&
		retainTokenSize &&
		retainReceiptId &&
		retainTotalQuacks &&
		retainAssetAmounts &&
		retainErgAmounts 
	}
}
```
