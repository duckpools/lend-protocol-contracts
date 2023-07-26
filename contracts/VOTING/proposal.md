```scala
{
	val treasuryNft = fromBase58("")
	val currentScript = SELF.propositionBytes
	val currentTokens = SELF.tokens(0)
	val currentProportion = SELF.R4[Long].get
	val currentRecipient = SELF.R5[Coll[Byte]].get
	val currentValidationHeight = SELF.R6[Long].get
	
	val countingBox = CONTEXT.dataInputs(0)
	val countingBoxTokens = countingBox.tokens(0)
	
	val isValidCountingBox = (
		countingBoxTokens._1 == currentTokens._1 && 
		countingBoxTokens._2 == currentValidationHeight
	)
	
	val successor = OUTPUTS(0)
	val successorScript = successor.propositionBytes
	val successorTokens = successor.tokens(0)
	val successorProportion = successor.R4[Long].get
	val successorRecipient = successor.R5[Coll[Byte]].get
	
	val isValidScript = successorScript == currentScript
	val isValidTokens = successorTokens._1 == currentTokens._1 && successorTokens._2 == 2L
	val isProportionMaintained = successorProportion == currentProportion
	val isRecipientMaintained = successorRecipient == currentRecipient
	val isFirstUpdate = currentTokens._2 == 1
	
	if (CONTEXT.dataInputs.size == 1) {	
		isValidCountingBox &&
		isValidScript &&
		isProportionMaintained &&
		isRecipientMaintained &&
		isFirstUpdate
	} else {
		val treasury = CONTEXT.dataInputs(1)
		treasury.tokens(0)._1 == treasuryNft &&
		OUTPUTS.forall{
			(out : Box) => out.tokens.forall{
				(token : (Coll[Byte], Long)) => token._1 != currentTokens._1
			}
		}
	}	
}
```
