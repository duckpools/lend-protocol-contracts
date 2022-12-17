```scala
{
	val InitiallyLockedLP    = 9000000001000000L // Set 1000000 higher than arbitary max LP value such that initial lend token value is 1 nanoErg 
	val collateralBoxScript  = fromBase58("2mPHzyWLuq8yxUdsmCSzAYAyveFtvWhtqNyBybKEkkqE")
	val sigUSDTokenId        = fromBase58("GYATox71P9XAERmzoDdTGELa62f5ALyjxJLRSfJfKsh")
	val sUsdErgDexPoolNFT    = fromBase58("BJbaZAXMoFm9gi2MBXA9eyPi38ugjKZ66SQrnwQmoDNj")
	val sigRsvTokenId        = fromBase58("1uuSELXj5meePwVr5PX1ndCSEsUvqV4EmNunatgho9y")
	val sRsvErgDexPoolNFT    = fromBase58("2ybHuZL1EdKsc84KaKgKsbQn9KiL7Py8PWedunbrc7dx")
	val interestBoxNFT       = fromBase58("Gne6mUgPuJmzATwkNCpMyZp173K76dQEC9jMTGpDvP3p")
	val collateralMultiplier = 13 // To be divided by 10, so rate is 1.3
	val collateralDenom      = 10
	val MinValue             = 1000000 // this many number of nanoErgs are going to be permanently locked
	val decimalMultiplier    = 10000000L
	val minLoanValue         = 50000000L
	
	val poolNFT0    = SELF.tokens(0)
	val reservedLP0 = SELF.tokens(1)
	val borrow0     = SELF.R4[Long].get

	val successor = OUTPUTS(0)

	val poolNFT1    = successor.tokens(0)
	val reservedLP1 = successor.tokens(1)
	val borrow1     = successor.R4[Long].get

	val validSuccessorScript = successor.propositionBytes == SELF.propositionBytes

	val preservedPoolNFT     = poolNFT1 == poolNFT0
	val validLP              = reservedLP1._1 == reservedLP0._1
	// since tokens can be repeated, we ensure for sanity that there are no more tokens
	val noMoreTokens         = successor.tokens.size == 2
	val validMinValue        = successor.value >= MinValue

	val supplyLP0 = InitiallyLockedLP - reservedLP0._2
	val supplyLP1 = InitiallyLockedLP - reservedLP1._2

	val heldErg0 = SELF.value
	val heldErg1 = successor.value

	val deltaBorrow = borrow1 - borrow0
	
	val validDeltaBorrow = deltaBorrow == 0

	val lpValue0 = (heldErg0 + borrow0) / supplyLP0
	val lpValue1 = (heldErg1 + borrow1) / supplyLP1

	val validLPValue = lpValue1 >= lpValue0 // Ensures the current value of an LP token has not decreased

	val validExchange = (
		validSuccessorScript &&
		preservedPoolNFT &&
		validLP &&
		validDeltaBorrow &&
		validLPValue &&
		noMoreTokens &&
		validMinValue
	)

	// Also allow for deposits
	
	val deltaErg = heldErg1 - heldErg0
	val deltaLP  = reservedLP1._2 - reservedLP0._2 

	val validDeltaErg           = deltaErg > 0
	val validDeltaLp            = deltaLP == 0
	val validDeltaBorrowDeposit = deltaErg >= deltaBorrow * - 1	|| (borrow0 - deltaErg < 0  && borrow1 == 0) // this ensures LP value cannot decrease
	val minBorrowValue = borrow1 >= 0 // ensures delta borrow cannot be negative
	
	val validDeposit = (
		validSuccessorScript &&
		preservedPoolNFT &&
		validDeltaErg &&
		validLP &&
		validDeltaLp &&
		noMoreTokens &&
		validDeltaBorrowDeposit &&
		minBorrowValue
	)

	if (CONTEXT.dataInputs.size > 0) {	
		// Allow for loans

		val collateralBox    = OUTPUTS(1)
		val dexBox           = CONTEXT.dataInputs(0)
		val interestBox      = CONTEXT.dataInputs(1)

		val borrowAmount = collateralBox.R4[Long].get

		val ergInDexPool    = dexBox.value 
		val tokenInDexPool  = dexBox.tokens(2)._2
		val nanoErgsPerToken = ergInDexPool / tokenInDexPool
		val ergToTokenPrice = tokenInDexPool * decimalMultiplier / ergInDexPool

		val interestHistory = interestBox.R4[Coll[Long]].get

		val validCollateralBoxScript = blake2b256(collateralBox.propositionBytes) == collateralBoxScript
		val validInterestBox         = interestBox.tokens(0)._1 == interestBoxNFT


		val validCollateral = if (dexBox.tokens(0)._1 == sUsdErgDexPoolNFT) {
			// sigUsd as collateral
			val correctToken  = if (collateralBox.tokens.size != 0) {collateralBox.tokens(0)._1 == sigUSDTokenId} else {false} 
			val correctAmount = if (collateralBox.tokens.size != 0) {collateralBox.tokens(0)._2 >= (borrowAmount * collateralMultiplier) / (nanoErgsPerToken * collateralDenom)} else {false}
			correctToken && correctAmount
		} else if (dexBox.tokens(0)._1 == sRsvErgDexPoolNFT) {
			// sigRsv as collateral
			val correctToken  = if (collateralBox.tokens.size != 0) {collateralBox.tokens(0)._1 == sigRsvTokenId} else {false} 
			val correctAmount = if (collateralBox.tokens.size != 0) {collateralBox.tokens(0)._2 >= (borrowAmount * collateralMultiplier) / (nanoErgsPerToken * collateralDenom)} else {false}
			correctToken && correctAmount
		} else {
			false
		}
		val validInterestIndex   = collateralBox.R6[Int].get == interestHistory.size - 1

		val validLendPoolErg = deltaErg * -1 == borrowAmount
		val validBorrowDelta = deltaBorrow == borrowAmount
		
		val validMinLoanValue = borrowAmount >= minLoanValue
		
		val validLendPoolTokens = successor.tokens == SELF.tokens
		
		val validCollateralBoxRegisters = collateralBox.R5[Coll[Byte]].isDefined // to prevent compilation error for collateral box

		val validBorrow = (
			validSuccessorScript &&
			validLendPoolErg &&
			validLendPoolTokens &&
			validBorrowDelta &&
			validCollateralBoxScript &&
			validCollateralBoxRegisters &&
			validInterestBox &&
			validCollateral &&
			validInterestIndex &&
			validMinLoanValue
		) 
		sigmaProp(validBorrow) 
	} else {
		sigmaProp(validDeposit || validExchange)
	}	
}
```
