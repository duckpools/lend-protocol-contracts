```scala
{
	val counterToken = fromBase58("")
	val minimumCounterTokens = 100000000
	
	val userPk = SELF.R6[GroupElement].get
	val userTree = SELF.R7[Coll[Byte]].get
	val recordedSubmission = SELF.R8[Long].get
	
	if (CONTEXT.dataInputs.size > 0) {
		val counterBox = CONTEXT.dataInputs(0)
		val isValidCounterBox = counterBox.tokens(0)._1 == counterToken && counterBox.tokens(0)._2 > minimumCounterTokens
		val counterDeadline = counterBox.R4[Long].get
		
		sigmaProp(
			proveDlog(userPk) &&
			HEIGHT < counterDeadline &&
			HEIGHT > recordedSubmission + 100 &&
			OUTPUTS.forall{
				(out : Box) => out.tokens.forall{
					(token : (Coll[Byte], Long)) => token._1 != SELF.tokens(0)._1
				}
			}
		)
	} else {
		val counterBox = INPUTS(0)
		val isValidCounterBox = counterBox.tokens(0)._1 == counterToken && counterBox.tokens(0)._2 > minimumCounterTokens
		
		val userGetsTokensBack = OUTPUTS.slice(1, OUTPUTS.size).exists {
			(out : Box) => (
				out.propositionBytes == userTree &&
				out.tokens(0) == SELF.tokens(1) &&
				out.R4[Coll[Byte]].get == SELF.id
			)
		}
		
		sigmaProp(
			isValidCounterBox &&
			userGetsTokensBack 
		)
	}
}
```
