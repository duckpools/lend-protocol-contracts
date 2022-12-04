```scala
{
	val txFee   = 1000000L
	val poolNFT = fromBase58("GGbKYdk2Qk5tXNYoCBTjMYeKxK5bBZ42raqWv5FXDS52")
	
	val initalPool = INPUTS(0)
	val finalPool  = OUTPUTS(0)
	
	val loanAmount = min(SELF.R4[Long].get, SELF.value - txFee)
	
	val borrow0 = initalPool.R4[Long].get
	val borrow1 = finalPool.R4[Long].get
	
	val deltaBorrowed = borrow0 - borrow1 // Amount borrowed will reduce
	
	val validFinalPool = finalPool.tokens(0)._1 == poolNFT
	val validInitialPool = initalPool.tokens(0)._1 == poolNFT
	
	val deltaValue = finalPool.value - initalPool.value
	
	val validValue    = deltaValue >= SELF.value - txFee
	val validTokens   = finalPool.tokens == initalPool.tokens
	val validBorrowed = deltaBorrowed == loanAmount || (borrow0 - loanAmount < 0  && borrow1 == 0)
	
	val multiBoxSpendSafety = INPUTS.size == 2
	
	sigmaProp(
		validFinalPool &&
		validInitialPool &&
		validValue &&
		validTokens &&
		validBorrowed &&
		multiBoxSpendSafety
	)
}
```
