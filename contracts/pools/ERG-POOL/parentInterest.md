```scala
{
	// Constants
	val MaximumExecutionFee = 5000000
	val ChildExecutionFee = 2000000
	val InterestContractDenomination = 100000000L
	val MaxHistorySize = 100

	// Get current box values
	val currentScript = SELF.propositionBytes
	val currentValue = SELF.value
	val currentParentToken = SELF.tokens(0)
	val currentChildTokens = SELF.tokens(1)
	val currentInterestHistory = SELF.R4[Coll[Long]].get

	// Get successor box values
	val successor = OUTPUTS(0)
	val successorScript = successor.propositionBytes
	val successorValue = successor.value
	val successorParentToken = successor.tokens(0)
	val successorChildTokens = successor.tokens(1)
	val successorInterestHistory = successor.R4[Coll[Long]].get

	// Get header child values
	val headChild = CONTEXT.dataInputs(0)
	val headChildScript = headChild.propositionBytes
	val headChildToken = headChild.tokens(0)
	val headInterestHistory = headChild.R4[Coll[Long]].get
	val headChildHeight = headChild.R5[Long].get
	val headIndex = headChild.R6[Int].get
	
	// Get new child values
	val newChild = OUTPUTS(1)
	val newChildScript = newChild.propositionBytes
	val newChildValue = newChild.value
	val newChildToken = newChild.tokens(0)
	val newInterestHistory = newChild.R4[Coll[Long]].get
	val newChildHeight = newChild.R5[Long].get
	val newIndex = newChild.R6[Int].get

	// Validate successor parent
	val validSuccessorScript = successorScript == currentScript
	val validSuccessorValue = successorValue >= currentValue - MaximumExecutionFee - (MaxHistorySize * ChildExecutionFee)
	val retainParentToken = successorParentToken == currentParentToken
	val validChildTokensId = successorChildTokens._1 == currentChildTokens._1 
	val validChildTokensAmount = successorChildTokens._2 == currentChildTokens._2 - 1

	// Calculate successorInterestHistory
	val totalInterestAccrued = headInterestHistory.fold(InterestContractDenomination, {(z:Long, base:Long) => (z * base / InterestContractDenomination)})
	val validSuccessorInterestHistory = (
		currentInterestHistory.append(Coll(totalInterestAccrued)) == successorInterestHistory
	)
	
	// Validate head child
	val validHeaderTokenId = headChildToken._1 == currentChildTokens._1
	val validHeaderIndex = headIndex == currentInterestHistory.size
	val validHeaderHistorySize = headInterestHistory.size >= MaxHistorySize
	
	// Validate new child
	val validChildScript = newChildScript == headChildScript
	val validChildValue = newChildValue >= MaxHistorySize * ChildExecutionFee
	val validChildToken = newChildToken == headChildToken
	val validChildIndex = newIndex == successorInterestHistory.size
	val validChildHistory = newInterestHistory == Coll(InterestContractDenomination)
	val validChildHeight = newChildHeight == headChildHeight
	
	sigmaProp(
		validSuccessorScript &&
		validSuccessorValue &&
		retainParentToken &&
		validChildTokensId &&
		validChildTokensAmount &&
		validSuccessorInterestHistory &&
		validHeaderTokenId &&
		validHeaderIndex &&
		validHeaderHistorySize &&
		validChildScript &&
		validChildValue &&
		validChildToken &&
		validChildIndex &&
		validChildHistory &&
		validChildHeight && true && HEIGHT > 1
	)
}
```
