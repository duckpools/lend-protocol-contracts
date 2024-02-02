```scala
{
	val collateralBoxScript  = fromBase58("YYfa7bEzKR8LYzCbQbHQRM3s7JgRouqB2qzvKy9nWyZ")
	val minTxFee      = 1000000L
	val minBoxValue   = 1000000L
	val poolNFT       = fromBase58("8A5R3bhY16TCJXPF8cCNL5bgdBg1vHZyM3f8W1PqAvdM")
 	val BorrowTokenId = fromBase58("2z3Fmiezj1EzsaLXU4bmCFRkEkhiEcuhZ7dQTdERaeqC")
	val PoolNativeCurrency = fromBase58("GYATox71P9XAERmzoDdTGELa62f5ALyjxJLRSfJfKsh")
	
	val user          = SELF.R4[Coll[Byte]].get
	val requestAmount = SELF.R5[Long].get
	val publicRefund  = SELF.R6[Int].get
	val userThresholdPenalty = SELF.R7[(Long, Long)].get
	val userDexNft = SELF.R8[Coll[Byte]].get
	val userPk = SELF.R9[GroupElement].get
	
	val operation = if (OUTPUTS.size < 3) {
		val refundBox = OUTPUTS(0)
		val deltaErg = SELF.value - refundBox.value
		
		val validRefundRecipient = refundBox.propositionBytes == user
		val multiBoxSpendRefund = refundBox.R4[Coll[Byte]].get == SELF.id
		val validDeltaErg = deltaErg <= minTxFee
		val validHeight   = HEIGHT >= publicRefund		
		
		val refund = (
			validRefundRecipient  &&
			validDeltaErg &&
			multiBoxSpendRefund &&
			validHeight 
		)
		refund	
	} else {
		val poolBox       = OUTPUTS(0)
		val collateralBox = OUTPUTS(1)
		val userBox       = OUTPUTS(2)
		
		val collateralTokens = collateralBox.tokens
		val collateral = collateralBox.value
		val collateralBorrowTokens = collateralBox.tokens(0)
		val recordedBorrower = collateralBox.R4[Coll[Byte]].get
		val indexes = collateralBox.R5[(Int, Int)].get
		val thresholdPenalty = collateralBox.R6[(Long, Long)].get
		val dexNft = collateralBox.R7[Coll[Byte]].get
		val collateralUserPk = collateralBox.R8[GroupElement].get
		val collateralForcedLiquidation = collateralBox.R9[(Long, Long)].get._1
		
		val loanAmount = collateralBorrowTokens._2
		
		val validCollateralBoxScript = blake2b256(collateralBox.propositionBytes) == collateralBoxScript
		val validCollateralTokens = collateral == SELF.value - minBoxValue - minTxFee
		val validLoanAmount = (
			loanAmount == userBox.tokens(0)._2 &&
			collateralBorrowTokens._1 == BorrowTokenId &&
			userBox.tokens(0)._1 == PoolNativeCurrency
			)
		
		val validBorrower = collateralBox.R4[Coll[Byte]].get == user
		val validThresholdPenalty = userThresholdPenalty == thresholdPenalty
		val validDexNFT = userDexNft == dexNft
		val validUserPk = userPk == collateralUserPk
		val validForcedLiquidation = collateralForcedLiquidation > HEIGHT + 65480 && collateralForcedLiquidation <= HEIGHT + 65520
		
		val validInterestIndex = INPUTS(0).tokens(0)._1 == poolNFT // enforced by pool contract
		
		val validUserScript = userBox.propositionBytes == user
		val validUserLoanAmount = userBox.tokens(0)._2 == requestAmount	
		val multiBoxSpendSafety = userBox.R4[Coll[Byte]].get == SELF.id
		
		val exchange = (
			validCollateralBoxScript &&
			validUserScript &&
			validCollateralTokens &&
			validLoanAmount &&
			validBorrower &&
			validThresholdPenalty &&
			validDexNFT &&
			validUserPk &&
			validForcedLiquidation &&
			validInterestIndex &&
			validUserLoanAmount && 
			multiBoxSpendSafety
		)
		exchange
	}
	operation || proveDlog(userPk)
}
```
