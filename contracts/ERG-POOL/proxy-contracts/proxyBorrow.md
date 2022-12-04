```scala
{
	val collateralBoxScript  = fromBase58("2mPHzyWLuq8yxUdsmCSzAYAyveFtvWhtqNyBybKEkkqE")
	val minTxFee      = 1000000L
	val minBoxValue   = 1000000L
	val poolNFT       = fromBase58("GGbKYdk2Qk5tXNYoCBTjMYeKxK5bBZ42raqWv5FXDS52")
	
	val user          = SELF.R4[Coll[Byte]].get
	val requestAmount = SELF.R5[Long].get
	val publicRefund  = SELF.R6[Int].get
	
	val poolBox       = OUTPUTS(0)
	val collateralBox = OUTPUTS(1)
	val userBox       = OUTPUTS(2)
	
	val collateralTokens = collateralBox.tokens
	
	val validCollateralBoxScript = blake2b256(collateralBox.propositionBytes) == collateralBoxScript
	val validUserScript          = userBox.propositionBytes == user
	
	val validCollateralTokens = collateralTokens(0) == SELF.tokens(0)
	
	val validLoanAmount    = collateralBox.R4[Long].get == userBox.value
	val validBorrower      = collateralBox.R5[Coll[Byte]].get == user
	val validInterestIndex = INPUTS(0).tokens(0)._1 == poolNFT // enforced by pool contract
	
	val validUserValue = userBox.value == requestAmount
	
	val multiBoxSpendSafety = userBox.R4[Coll[Byte]].get == SELF.id
	
	val exchange = (
		validCollateralBoxScript &&
		validUserScript &&
		validCollateralTokens &&
		validLoanAmount &&
		validBorrower &&
		validInterestIndex &&
		validUserValue && 
		multiBoxSpendSafety
	)
	
	val deltaErg = SELF.value - userBox.value
	
	val validDeltaErg = deltaErg <= minTxFee + minBoxValue + minBoxValue
	val validHeight   = HEIGHT >= publicRefund
	
	val validTokens = if(userBox.tokens.size != 0) {userBox.tokens(0)._1 == SELF.tokens(0)._1 && userBox.tokens(0)._2 == SELF.tokens(0)._2} else {false}
	
	val validDummyBoxScripts = poolBox.propositionBytes == user && collateralBox.propositionBytes == user
	
	val refund = (
		validUserScript  &&
		validDeltaErg &&
		multiBoxSpendSafety &&
		validHeight &&
		validTokens &&
		validDummyBoxScripts
	)
	
	sigmaProp(exchange || refund) 
}
```
