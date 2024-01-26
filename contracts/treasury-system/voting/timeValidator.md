```scala
{
	// Constants
	val userVoteContract = fromBase58("6dshN62KwVWd2zAKyM1ddESU2quNKzEiSEPsdz3Z1KLP")
	val quacksId = fromBase58("81KtBNiSkgB23BHv7xPrpnjjpj1CYWegbtMaGdRNTeR5")
	
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
	val isValidVoteTokens = userVoteTokens._1 == currentTokens._1 && userVoteTokens._2 == 1
	val isValidVoteHeight = userVoteHeight > HEIGHT
	
	val isScriptMaintained = SELF.propositionBytes == successor.propositionBytes
	val isTokensValid = successorTokens._1 == currentTokens._1 && successorTokens._2 == currentTokens._2 - 1
	
	isValidVoteScript &&
	isValidVoteTokens &&
	isValidVoteHeight &&
	isScriptMaintained &&
	isQuacksPresent &&
	isTokensValid
}
```
