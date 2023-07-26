```scala
{
		val userVoteTree = fromBase58("")
		
		val currentTokens = SELF.tokens(0)
		
		val successor = OUTPUTS(0)
		val successorTokens = successor.tokens(0)
		
		val userVote = OUTPUTS(1)
		val userVoteScript = userVote.propositionBytes
		val userVoteTokens = userVote.tokens(0)
		val userVoteHeight = userVote.R8[Long].get
		
		val isValidVoteScript = userVoteScript == userVoteTree
		val isValidVoteTokens = currentTokens._1 == userVoteTokens._1 && userVoteTokens._2 == 1
		val isValidVoteHeight = userVoteHeight > HEIGHT
		
		val isScriptMaintained = SELF.propositionBytes == successor.propositionBytes
		val isTokensValid = successorTokens.1 == currentTokens._1 && successorTokens._2 == currentTokens._2 - 1
		
		isValidVoteScript &&
		isValidVoteTokens &&
		isValidVoteHeight &&
		isScriptMaintained &&
		isTokensValid
}
