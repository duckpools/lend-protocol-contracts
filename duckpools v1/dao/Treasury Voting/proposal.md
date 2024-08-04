```scala
{
	val treasuryNft = fromBase58("Goj3WUfmRmWgF6AngvFowK6CwQbZTmapDY3H136f528K") // Set at Genesis
	val currentTokens = SELF.tokens(0)
	
	if (OUTPUTS(1).tokens.size == 1) {	
		// Load Current Values
		val currentScript = SELF.propositionBytes
		val currentProportion = SELF.R4[Long].get
		val currentRecipient = SELF.R5[Coll[Byte]].get
		val currentValidationHeight = SELF.R6[Long].get
		
		// Load Counting Box
		val countingBox = INPUTS(0)
		val countingBoxTokens = countingBox.tokens(0)
		
		val isValidCountingBox = (
			countingBoxTokens._1 == currentTokens._1 && 
			countingBoxTokens._2 == currentValidationHeight
		)
		
		// Load Successor Values
		val successor = OUTPUTS(1)
		val successorScript = successor.propositionBytes
		val successorProportion = successor.R4[Long].get
		val successorRecipient = successor.R5[Coll[Byte]].get
	
		val successorTokens = successor.tokens(0)
		
		val isValidScript = successorScript == currentScript
		val isProportionMaintained = successorProportion == currentProportion
		val isRecipientMaintained = successorRecipient == currentRecipient
		val isFirstUpdate = currentTokens._2 == 1
		val isValidTokens = successorTokens._1 == currentTokens._1 && successorTokens._2 == 2L
		
		isValidCountingBox &&
		isValidScript &&
		isProportionMaintained &&
		isRecipientMaintained &&
		isFirstUpdate && 
		isValidTokens &&
		HEIGHT > 124
	} else {
		val treasury = INPUTS(1)
		treasury.tokens(0)._1 == treasuryNft &&
		OUTPUTS.forall{
			(out : Box) => out.tokens.forall{
				(token : (Coll[Byte], Long)) => token._1 != currentTokens._1
			}
		}
	}	
}			
```
