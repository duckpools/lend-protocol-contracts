```scala
{
	val txFee   = 1000000L
  val MaxBorrowTokens        = 9000000000000000L
	val poolNFT = fromBase58("2rEBTtAM81L3PghVyCwCccyh49EXGhSh3n2kLGufMTqe")
	
	val initalPool = INPUTS(0)
	val finalPool  = OUTPUTS(0)
	
	val loanAmount = SELF.tokens(0)._2
	
	val borrow0 = MaxBorrowTokens - initalPool.tokens(2)._2
	val borrow1 = MaxBorrowTokens - finalPool.tokens(2)._2
	
	val deltaBorrowed = borrow0 - borrow1 // Amount borrowed will reduce
	
	val validFinalPool = finalPool.tokens(0)._1 == poolNFT
	val validInitialPool = initalPool.tokens(0)._1 == poolNFT
	
	val deltaValue = finalPool.value - initalPool.value
	
	val validValue    = deltaValue >= SELF.value - txFee
	val validBorrowed = deltaBorrowed == loanAmount
	
	val multiBoxSpendSafety = INPUTS.size == 2
	
	sigmaProp(
		validFinalPool &&
		validInitialPool &&
		validValue &&
		validBorrowed &&
		multiBoxSpendSafety
	)
}
```
