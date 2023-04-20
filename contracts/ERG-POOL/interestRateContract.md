```scala
{
	val poolNFT                = fromBase58("2rEBTtAM81L3PghVyCwCccyh49EXGhSh3n2kLGufMTqe")
	val minBoxValue            = 1000000
	val InterestMultiplier     = 1000000
	val BootstrapTVLDifference = 999999
	val InitiallyLockedLP      = 9000000000000000L
  	val MaxBorrowTokens        = 9000000000000000L
	
	val successor = OUTPUTS(0)
	val pool      = CONTEXT.dataInputs(0)
	
	val interestHistory      = SELF.R4[Coll[Long]].get
	val recordedHeight       = SELF.R5[Long].get
	val finalInterestHistory = successor.R4[Coll[Long]].get
	val finalHeight          = successor.R5[Long].get
	
	val deltaHeight      = HEIGHT - recordedHeight
	val validDeltaHeight = deltaHeight >= 360 // About 12 hours
	
	val deltaFinalHeight      = finalHeight - HEIGHT
	val validDeltaFinalHeight = (deltaFinalHeight >= 0 && deltaFinalHeight <= 30)
	val supplyLP    = InitiallyLockedLP - pool.tokens(0)._2
	val borrowed    = MaxBorrowTokens - pool.tokens(2)._2
	val util        = InterestMultiplier * borrowed / (pool.value + borrowed)
	val currentRate = Coll(InterestMultiplier + (5000 + util * 13 / 100) / 365) 
	
	val retainedERG          = successor.value >= SELF.value
	val preservedInterestNFT = successor.tokens(0)._1 == SELF.tokens(0)._1
	
	val validSuccessorScript = SELF.propositionBytes == successor.propositionBytes
	val validInterestUpdate  = interestHistory.append(currentRate) == finalInterestHistory
	
	val validPoolBox = pool.tokens(0)._1 == poolNFT

	sigmaProp(
		retainedERG &&
		validSuccessorScript &&
		validInterestUpdate &&
		validPoolBox &&
		preservedInterestNFT &&
		validDeltaHeight &&
		validDeltaFinalHeight
	)
}
```
