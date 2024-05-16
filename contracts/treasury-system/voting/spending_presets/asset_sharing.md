```scala
{
	val emissionBlock = 1272551 
	val blocksInThreeMonths = 65700L
	val precision = 1000000000L
	val MaxReceiptTokens = 20000000000000L
		
	// Initial State
	val iScript = SELF.propositionBytes
	val iValue = SELF.value
	val iTokens = SELF.tokens
	val iReceiptTokens = SELF.tokens(0)
	val iQuacks = SELF.tokens(1)

	// Final State
	val successor = OUTPUTS(0)
	val fScript = successor.propositionBytes
	val fValue = successor.value
	val fTokens = successor.tokens
	val fReceiptTokens = successor.tokens(0)
	val fQuacks = successor.tokens(1)
	
	val deltaReceiptTokens = fReceiptTokens._2 - iReceiptTokens._2
	val deltaQuacks = fQuacks._2 - iQuacks._2

	val generalLogic = if (HEIGHT < emissionBlock) {
		// Allow Exchanges
		val isValidExchange = deltaQuacks == -1 * deltaReceiptTokens 
		val isDeltaQuacksPositive = deltaQuacks > 0
		
		val retainScript = iScript == fScript
		val retainValue = iValue == fValue
		val retainTokens = iTokens.slice(2, iTokens.size) == fTokens.slice(2, iTokens.size)
		val retainTokenSize = iTokens.size == fTokens.size
		val retainReceiptId = iReceiptTokens._1 == fReceiptTokens._1
		val retainQuacksId = iQuacks._1 == fQuacks._1 
		
		isValidExchange &&
		isDeltaQuacksPositive &&
		retainScript &&
		retainValue &&
		retainTokens &&
		retainTokenSize &&
		retainReceiptId &&
		retainQuacksId
	} else {
		// Allow withdrawals
		val circulatingReceipts = MaxReceiptTokens - iReceiptTokens._2
		val userShare = ((deltaReceiptTokens.toBigInt * precision.toBigInt).toBigInt / circulatingReceipts.toBigInt).toBigInt
		val isValidValue = fValue.toBigInt >= iValue.toBigInt - (userShare * iValue.toBigInt) / precision.toBigInt
		val slicedExpectedTokens = iTokens.slice(1, iTokens.size)
		val expectedTokens = slicedExpectedTokens.map{
				(token : (Coll[Byte], Long)) => 
				(token._1, max((token._2.toBigInt -  (userShare * token._2.toBigInt) / precision.toBigInt), 1.toBigInt))
		}
		
		val outputTokensBigInt = fTokens.slice(1, iTokens.size).map{
				(token : (Coll[Byte], Long)) => 
				(token._1, token._2.toBigInt)
		}
		val isValidTokens = expectedTokens == outputTokensBigInt
		val isDeltaReceiptsPositive = deltaReceiptTokens > 0
		
		val retainScript = iScript == fScript
		val retainTokenSize = iTokens.size == fTokens.size
		val retainReceiptId = iReceiptTokens._1 == fReceiptTokens._1
		
		isValidValue &&
		isValidTokens &&
		isDeltaReceiptsPositive &&
		retainScript &&
		retainTokenSize &&
		retainReceiptId
	}
	sigmaProp(generalLogic) || (PK("") && HEIGHT > (emissionBlock + blocksInThreeMonths))
}
```
