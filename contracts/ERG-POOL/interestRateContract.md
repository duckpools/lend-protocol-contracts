```scala
{
    val poolNFT                = fromBase58("2rEBTtAM81L3PghVyCwCccyh49EXGhSh3n2kLGufMTqe")
    val minBoxValue            = 1000000
    val InterestMultiplier     = 1000000
    val coefficientDenomination = 100
    val InitiallyLockedLP      = 9000000000000000L
    val MaxBorrowTokens        = 9000000000000000L
    val ParamaterBoxNft = fromBase58("9M8VueZ5srjdUmi9T2sfkHCZfQPabFbr6EkEiBfP7LWn")
    
    val successor = OUTPUTS(0)
    val pool      = CONTEXT.dataInputs(0)
    val parameterBox = CONTEXT.dataInputs(1)
    
    val interestHistory      = SELF.R4[Coll[Long]].get
    val recordedHeight       = SELF.R5[Long].get
    val finalInterestHistory = successor.R4[Coll[Long]].get
    val finalHeight          = successor.R5[Long].get
    
    // get coefficients
    val coefficients = parameterBox.R4[Coll[Long]].get
    val a = coefficients(0)
    val b = coefficients(1)
    val c = coefficients(2)
    val d = coefficients(3)
    val e = coefficients(4)
    val f = coefficients(5)
     
    
    val deltaHeight      = HEIGHT - recordedHeight
    val validDeltaHeight = deltaHeight >= 360 // About 12 hours
    
    val deltaFinalHeight      = finalHeight - HEIGHT
    val validDeltaFinalHeight = (deltaFinalHeight >= 0 && deltaFinalHeight <= 30)
    val supplyLP    = InitiallyLockedLP - pool.tokens(0)._2
    val borrowed    = MaxBorrowTokens - pool.tokens(2)._2
    val util        = InterestMultiplier * borrowed / (pool.value + borrowed)
    
    val D = coefficientDenomination
    val M = InterestMultiplier
    val x = util
    
    val currentRate = (
        M + (
            a + 
            (b * x) / D + 
            (c * x) / D * x / M +
            (d * x) / D * x / M * x / M +
            (e * x) / D * x / M * x / M * x / M + 
            (f * x) / D * x / M * x / M * x / M * x / M
            )
        ) 
    
    val retainedERG          = successor.value >= SELF.value
    val preservedInterestNFT = successor.tokens(0)._1 == SELF.tokens(0)._1
    
    val validSuccessorScript = SELF.propositionBytes == successor.propositionBytes
    val validInterestUpdate  = interestHistory.append(Coll(currentRate)) == finalInterestHistory
    
    val validPoolBox = pool.tokens(0)._1 == poolNFT
    val validParameterBox = parameterBox.tokens(0)._1 == ParamaterBoxNft

    sigmaProp(
      retainedERG &&
      validSuccessorScript &&
      validInterestUpdate &&
      validPoolBox &&
      validParameterBox &&
      preservedInterestNFT &&
      validDeltaHeight &&
      validDeltaFinalHeight
    )
}
```
