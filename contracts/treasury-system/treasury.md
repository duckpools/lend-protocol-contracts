```scala
{
	// Constants
	val voteNft = fromBase58("4CQ77DnpkJS6iyEw58LWPS6F2a1WdshDo36TBLPx172W")
	val proportionDenomination = 10000000L
	
	// Branch to seperate deposit and withdrawal
	if (INPUTS(0) == SELF) {
		// deposit
		val successorTreasury = OUTPUTS(0)
		val isSuccessorScriptValid = successorTreasury.propositionBytes == SELF.propositionBytes
		val isSuccessorValueValid = successorTreasury.value >= SELF.value
		val isSuccessorTokensIncreasing = SELF.tokens.forall {
			(iToken: (Coll[Byte], Long)) => successorTreasury.tokens.exists{
			(sToken: (Coll[Byte], Long)) => sToken._1 == iToken._1 && sToken._2 >= iToken._2
			}}	
		// Check if vote given to add more tokens, if so add more tokens.
		val isValidTokenSize = if (INPUTS(1).tokens.size > 0 && INPUTS(1).tokens(0)._1 == voteNft) {
			val tokensToAdd = INPUTS(1).R6[Coll[Coll[Byte]]].get
			tokensToAdd.forall { 
				(tokenId : Coll[Byte]) =>
					successorTreasury.tokens.exists {
						(token: (Coll[Byte], Long)) => token._1 == tokenId
				}
			} && SELF.tokens.size + tokensToAdd.size == successorTreasury.tokens.size 
		} else {
			SELF.tokens.size == successorTreasury.tokens.size
		}
		isSuccessorScriptValid &&
		isSuccessorValueValid &&
		isSuccessorTokensIncreasing &&
		isValidTokenSize 
	} else if (INPUTS(1) == SELF) {
		// Withdrawal
		// Read vote result
		val voteResult = INPUTS(0)
		val proportionAwared = voteResult.R4[Long].get
		val recipient = voteResult.R5[Coll[Byte]].get
		
		// Validate result box
		val isValidResult = voteResult.tokens(0)._1 == voteNft && voteResult(0)._2 == 2
		
		// Branch for regular withdrawal and new treasury withdrawal
		if (proportionAwared != proportionDenomination) {	
			// Regular withdrawal
			// Load output boxes and values
			val successorTreasury = OUTPUTS(0)
			val recipientBox = OUTPUTS(1)
			val valueAwarded = SELF.value * proportionAwared / proportionDenomination	
			val tokensToScan = SELF.tokens.slice(1,SELF.tokens.size)
		
			// Validate treasury recreation
			val isScriptRetained = SELF.propositionBytes == successorTreasury.propositionBytes
			val isValidValue = successorTreasury.value >= SELF.value - valueAwarded
			val isTreasuryNftRetained = SELF.tokens(0) == successorTreasury.tokens(0)
			// Validate tokens (non zero check not neccessary for treasury)
			val expectedTreasuryTokens = tokensToScan.map{
				(token : (Coll[Byte], Long)) => 
				(token._1, token._2 - token._2 * proportionAwared / proportionDenomination)
			}
			val isValidTreasuryTokens = successorTreasury.tokens.slice(1, successorTreasury.tokens.size) == expectedTreasuryTokens
			val isSameTokenSize = SELF.tokens.size == successorTreasury.tokens.size
			
			// Validate recipient box
			val isValidRecipient = recipientBox.propositionBytes == recipient
			val isValidRecipientValue = recipientBox.value >= valueAwarded
			
			// Validate recipient tokens
			// Only check tokens with expected amount > 0
			val nonZeroRecipientTokens = tokensToScan.filter {
				(token : (Coll[Byte], Long)) =>
				(token._2 * proportionAwared / proportionDenomination) > 0
			}	
			// Ensure output tokens has amount governed by proportionAwared
			val expectedRecipientTokens = nonZeroRecipientTokens.map{
				(token : (Coll[Byte], Long)) => 
				(token._1, token._2 * proportionAwared / proportionDenomination)
			}
			val isValidRecipientTokens = recipientBox.tokens == expectedRecipientTokens
			
			// Apply validations
			isValidResult &&
			isScriptRetained &&
			isValidValue &&
			isTreasuryNftRetained &&
			isValidTreasuryTokens &&
			isValidRecipient &&
			isValidRecipientValue &&
			isValidRecipientTokens &&
			isSameTokenSize
		} else {
			// New Treasury
			val newTreasury = OUTPUTS(0)
			
			val isValidNewScript = newTreasury.propositionBytes == recipient
			val isValidNewTreasuryValue = newTreasury.value >= SELF.value
			val isValidNewTreasuryTokens = newTreasury.tokens == SELF.tokens
			val isSameTokenSize = SELF.tokens.size == newTreasury.tokens.size
			
			isValidNewScript &&
			isValidNewTreasuryValue &&
			isValidNewTreasuryTokens &&
			isSameTokenSize
		}
	} else {
		false
	}
}
