```scala
{
	// Constants
	val userVoteContract = fromBase58("9bjVXo29fWDenQ1xvubpNinB5ZoZ5PdLhEUGWZr8Qudx")
	val quacksId = fromBase58("aa5Hq5V5ssGxbReLMzpJ55nVrb73CWJ1oRiKD2X5Qkv")
	val MinimumVoteAmount = 50000L
	
	val currentTokens = SELF.tokens(0)
	
	val successor = OUTPUTS(0)
	val successorTokens = successor.tokens(0)
	
	val userVote = OUTPUTS(1)
	val userVoteScript = userVote.propositionBytes
	val userVoteTokens = userVote.tokens(0)
	val userQuacks = userVote.tokens(1)
	val userVoteHeight = userVote.R8[Long].get
	
	val isValidVoteScript = blake2b256(userVoteScript) == userVoteContract
	val isQuacksPresent = userQuacks._1 == quacksId
	val isAboveMinimumVote = userQuacks._2 >= MinimumVoteAmount
	val isValidVoteTokens = userVoteTokens._1 == currentTokens._1 && userVoteTokens._2 == 1
	val isValidVoteHeight = userVoteHeight > HEIGHT
	
	val isScriptMaintained = SELF.propositionBytes == successor.propositionBytes
	val isTokensValid = successorTokens._1 == currentTokens._1 && successorTokens._2 == currentTokens._2 - 1
	
	isValidVoteScript &&
	isValidVoteTokens &&
	isValidVoteHeight &&
	isScriptMaintained &&
	isQuacksPresent &&
	isAboveMinimumVote &&
	isTokensValid
}
```
