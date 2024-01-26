demo indicative contract:

```scala
{
	val votesFor = SELF.R4[Long].get
	val deadline = SELF.R5[Long].get
	val proposal = SELF.R6[Coll[Byte]].get
	val title = SELF.R7[Coll[Byte]].get
	val placeHolderQuacks = SELF.tokens(0)
	val realQuacks = SELF.tokens(1)
	
	val finalVotesFor = OUTPUTS(0).R4[Long].get
	val finalDeadline = OUTPUTS(0).R5[Long].get
	val finalProposal = OUTPUTS(0).R6[Coll[Byte]].get
	val finalTitle = OUTPUTS(0).R7[Coll[Byte]].get
	val finalPlaceHolderQuacks = OUTPUTS(0).tokens(0)
	val finalRealQuacks = OUTPUTS(0).tokens(1)
	
	val deltaQuacks = finalRealQuacks._2 - realQuacks._2
	
	val retainScript = SELF.propositionBytes == OUTPUTS(0).propositionBytes
	val retainVotes = votesFor == finalVotesFor
	val retainDeadline = deadline == finalDeadline
	val retainProposal = proposal == finalProposal
	val retainTitle = title == finalTitle
	val retainTokenIds = placeHolderQuacks._1 == finalPlaceHolderQuacks._1 && finalRealQuacks._1 == realQuacks._1
	val correctExchange = placeHolderQuacks._2 - deltaQuacks == finalPlaceHolderQuacks._2
	
	sigmaProp(if (HEIGHT < deadline) {		
		val validVoteCount = (
			finalVotesFor == votesFor + deltaQuacks ||
			finalVotesFor == votesFor - deltaQuacks
		)
		
		val onlyAllowIncrease = deltaQuacks > 0
		
		retainScript &&
		validVoteCount &&
		retainDeadline &&
		retainProposal &&
		retainTitle &&
		retainTokenIds &&
		onlyAllowIncrease &&
		correctExchange
	} else {
		val onlyAllowDecrease = deltaQuacks < 0
	
		retainScript &&
		retainVotes &&
		retainDeadline &&
		retainProposal &&
		retainTitle &&
		retainTokenIds &&
		correctExchange &&
		onlyAllowDecrease
	}) || sigmaProp(prop && HEIGHT > value)
}
```
