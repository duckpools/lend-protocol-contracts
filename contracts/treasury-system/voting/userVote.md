```scala
{
	// Constants
	val counterToken = fromBase58("3jrqHvrN99UXKU8FZEh3HmX3uNs6G5EJj8nwA4MFmyWS")
	val minimumCounterTokens = 100000000 // In excess of 2 (to exclude proposal boxes)
	val cancellationCooldown = 20 // To Prevent spam on height validator
	
	// Fetch Registers
	val userPk = SELF.R6[GroupElement].get
	val userTree = SELF.R7[Coll[Byte]].get
	val recordedSubmission = SELF.R8[Long].get
	
	if (CONTEXT.dataInputs.size > 0) {
		// Fetch and validate counter box
		val counterBox = CONTEXT.dataInputs(0)
		val isValidCounterBox = counterBox.tokens(0)._1 == counterToken && counterBox.tokens(0)._2 > minimumCounterTokens
		val counterDeadline = counterBox.R4[Long].get
		
		
		// Require that the user has the private key corresponding to the public key stored in the box
		// And that the current block height is less than the deadline specified in the counter box
		// And that the current block height is greater than the recorded submission height + cooldown
		// And that vote token is burnt
		sigmaProp(
			proveDlog(userPk) &&
			HEIGHT < counterDeadline &&
			HEIGHT > recordedSubmission + cancellationCooldown &&
			OUTPUTS.forall{
				(out : Box) => out.tokens.forall{
					(token : (Coll[Byte], Long)) => token._1 != SELF.tokens(0)._1
				}
			}
		)
	} else {
		// Fetch and validate counter box
		val counterBox = INPUTS(0)
		val isValidCounterBox = counterBox.tokens(0)._1 == counterToken && counterBox.tokens(0)._2 > minimumCounterTokens
		
		// Check if the user gets the tokens back in the output boxes
		val userGetsTokensBack = OUTPUTS.slice(1, OUTPUTS.size).exists {
			(out : Box) => (
				out.propositionBytes == userTree &&
				out.tokens(0) == SELF.tokens(1) &&
				out.R4[Coll[Byte]].get == SELF.id
			)
		}
		// Require that the counter box is valid and the user gets the tokens back
		sigmaProp(
			isValidCounterBox &&
			userGetsTokensBack 
		)
	}
}
```