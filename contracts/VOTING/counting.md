```scala
{
	val nextVoteDeadline = SELF.R4[Long].get
	val passProposalDeadline = nextVoteDeadline + 150
	val newProposalDeadline = passProposalDeadline + 150
	val validVoteId = fromBase58("")
	val quacksId = fromBase58("")
	val minimumVotes = 100000L
	val voteResultDenomination = 1000L
	val minimumSupport = 500L
	val proposalTree = fromBase58("")
	val votingPeriodicity = 1000
	
	val currentScript = SELF.propositionBytes
	val currentValue = SELF.value
	val currentResultTokens = SELF.tokens(0)
	val currentProportionVote = SELF.R5[(Long, Long)].get
	val currentRecipientVote = SELF.R6[Coll[Byte]].get
	val currentTotalVotes = SELF.R7[Long].get
	val currentInitiationAmount = SELF.R8[Long].get
	val currentVoteValidation = SELF.R9[Long].get
	
	val successor = OUTPUTS(0)
	val successorScript = successor.propositionBytes
	val successorValue = successor.value
	val successorResultTokens = successor.tokens(0)
	val successorVoteDeadline = successor.R4[Long].get
	val successorProportionVote = successor.R5[(Long, Long)].get
	val successorRecipientVote = successor.R6[Coll[Byte]].get
	val successorTotalVotes = successor.R7[Long].get
	val successorInitiationAmount = successor.R8[Long].get
	val successorVoteValidation = successor.R9[Long].get
	
	val isBeforeCounting = HEIGHT < nextVoteDeadline
	val isCountingPeriod = HEIGHT > nextVoteDeadline && HEIGHT < passProposalDeadline
	val isVoteValidationPeriod = HEIGHT > passProposalDeadline && HEIGHT < newProposalDeadline
	val isNewProposalPeriod = HEIGHT > newProposalDeadline

	if (isBeforeCounting) {
		// Allow for updates to voting item
		val voteInitiationBox = CONTEXT.dataInputs(0)
		val votePower = voteInitiationBox.tokens(0)._2 
		val nominatedProportion = voteInitiationBox.R4[Long].get
		val nominatedRecipient = voteInitiationBox.R5[Coll[Byte]].get
		
		val isValidInitiationBox = (
			voteInitiationBox.tokens(0)._1 == quacksId && 
			votePower > currentInitiationAmount &&
			votePower > minimumVotes
		)
		
		// Recreate successor with new vote item
		val isValidScript = successorScript == currentScript 
		val isValidValue = successorValue >= currentValue
		val isTokensRetained = successor.tokens == SELF.tokens
		val isDeadlineMaintained = nextVoteDeadline == successorVoteDeadline
		val isProportionVoteValid = successorProportionVote._2 == 0L && successorProportionVote._1 == nominatedProportion
		val isRecipientVoteValid = successorRecipientVote == nominatedRecipient
		val isTotalVotesValid = successorTotalVotes == 0L
		val isSuccessorInitiationValid = successorInitiationAmount == votePower
		val isVoteValidationValid = successorVoteValidation == 0L
		
		isValidInitiationBox &&
		isValidScript &&
		isValidValue &&
		isTokensRetained &&
		isDeadlineMaintained &&
		isProportionVoteValid &&
		isRecipientVoteValid &&
		isTotalVotesValid &&
		isSuccessorInitiationValid &&
		isVoteValidationValid
	} else if (isCountingPeriod) {
		// Count votes that support sending some particular proportion of treasury
		// to some ergotree.
		val votesInFavour = INPUTS.slice(1,INPUTS.size).fold(0L, {
			(z:Long, base:Box) => if (base.R4[Long].get == currentProportionVote._1) {
				z + base.tokens(1)._2
			} else {
				z
			}
		})
		val totalVotes = INPUTS.slice(1,INPUTS.size).fold(0L, {
			(z:Long, base:Box) => z + base.tokens(1)._2
		})
		val validationVotesInFavour = INPUTS.slice(1,INPUTS.size).fold(0L, {
			(z:Long, base:Box) => if (base.R9[Int].get == 1) {
				z + base.tokens(1)._2
			} else {
				z - base.tokens(1)._2
			}
		})
		val expectedTotalVotes = totalVotes + currentTotalVotes 
		val expectedProportionVotes = currentProportionVote._2 + votesInFavour
		
		val isValidVotes = INPUTS.slice(1,INPUTS.size).forall{
			(in : Box) => (
				in.tokens(0)._1 == validVoteId &&
				in.tokens(1)._1 == quacksId &&
				in.R8[Long].get < nextVoteDeadline
		}
		
		// Recreate box with new vote counts
		val isValidScript = successorScript == currentScript 
		val isValidValue = successorValue >= currentValue
		val isTokensRetained = successor.tokens == SELF.tokens
		val isDeadlineMaintained = nextVoteDeadline == successorVoteDeadline
		val isProportionVoteValid = successorProportionVote._2 == expectedProportionVotes && successorProportionVote._1 == currentProportionVote._1
		val isRecipientVoteValid = successorRecipientVote == currentRecipientVote
		val isTotalVotesValid = successorTotalVotes == expectedTotalVotes
		val isVoteValidationValid = successorVoteValidation == currentVoteValidation + validationVotesInFavour
		
		// Vote count validations
		isValidScript &&
		isValidValue &&
		isTokensRetained &&
		isDeadlineMaintained &&
		isProportionVoteValid &&
		isRecipientVoteValid &&
		isTotalVotesValid &&
		isVoteValidationValid &&
		isValidVotes		
	} else if (isVoteValidationPeriod) {
		// Check votes to pass pending proposal
		val isVoteSuccessful = (
			currentVoteValidation > 0 &&
			currentTotalVotes > minimumVotes
		)
		val currentProposalBox = INPUTS(1)
		val currentProposalTokens = currentProposalBox.tokens(0)
		val currentProposalProportion = currentProposalBox.R4[Long].get
		val currentProposalRecipient = currentProposalBox.R5[Coll[Byte]].get
		val currentProposalValidationHeight = currentProposalBox.R6[Long].get
		
		val isValidProposalBox = (
			currentProposalTokens._1 == currentResultTokens._1 &&
			currentProposalTokens._2 == 1 &&
			currentResultTokens._2 == currentProposalValidationHeight  
		)
		
		val successorProposalBox = OUTPUTS(1)
		val successorProposalScript = successorProposalBox.propositionBytes
		val successorProposalValue = successorProposalBox.value
		val successorProposalNft = successorProposalBox.tokens(0)
		val successorProposalProportion = successorProposalBox.R4[Long].get
		val successorProposalRecipient = successorProposalBox.R5[Coll[Byte]].get
		
		// Construct proposal box
		val isValidProposalScript = successorProposalScript == proposalTree
		val isValidProposalNft = successorProposalNft._1 == currentResultTokens._1 && successorProposalNft._2 == 2
		val isValidProportion = currentProposalProportion == successorProposalProportion
		val isValidRecipient = currentProposalRecipient == successorProposalRecipient
		
		// Recreate box
		val isValidScript = successorScript == currentScript 
		val isValidValue = successorValue >= currentValue
		val isValidTokens = successorResultTokens._1 == currentResultTokens._1 && successorResultTokens._2 == currentResultTokens._2 - 1
		val isTokenArraySizeConstant = successor.tokens.size == SELF.tokens.size
		val isDeadlineMaintained = nextVoteDeadline == successorVoteDeadline
		val isVoteReset = successorProportionVote == (0L,0L)
		val isTotalVotesValid = successorTotalVotes == 0L
		val isVoteValidationValid = successorVoteValidation == 0L	
		
		isVoteSuccessful &&
		isValidProposalBox &&
		isValidProposalScript &&
		isValidProposalNft &&
		isValidProportion &&
		isValidRecipient &&
		isValidScript &&
		isValidValue &&
		isValidTokens &&
		isTokenArraySizeConstant &&
		isDeadlineMaintained &&
		isVoteReset &&
		isTotalVotesValid &&
		isVoteValidationValid
	} else if (isNewProposalPeriod) {
		// Write result of vote to a proposal box.
		val isVoteSuccessful = (
			currentProportionVote._2 * voteResultDenomination / currentTotalVotes > minimumSupport &&
			currentTotalVotes > minimumVotes
		)
		val proposalBox = OUTPUTS(1)
		val proposalScript = proposalBox.propositionBytes
		val proposalValue = proposalBox.value
		val proposalNft = proposalBox.tokens(0)
		val proposalProportion = proposalBox.R4[Long].get
		val proposalRecipient = proposalBox.R5[Coll[Byte]].get
		val proposalValidationHeight = proposalBox.R6[Long].get
		
		// Construct proposal box
		val isValidProposalScript = proposalScript == proposalTree
		val isValidProposalNft = proposalNft._1 == currentResultTokens._1 && proposalNft._2 == 1
		val isValidProportion = proposalProportion == currentProportionVote._2
		val isValidRecipient = proposalRecipient == currentRecipientVote
		val isValidValidationHeight = proposalValidationHeight == currentResultTokens._2
		
		// Recreate box
		val isValidScript = successorScript == currentScript 
		val isValidValue = successorValue >= currentValue
		val isValidTokens = successorResultTokens._1 == currentResultTokens._1 && successorResultTokens._2 == currentResultTokens._2 - 1
		val isTokenArraySizeConstant = successor.tokens.size == SELF.tokens.size
		val isNewDeadlineValid = successorVoteDeadline == nextVoteDeadline + votingPeriodicity
		val isVoteReset = successorProportionVote == (0L,0L)
		val isTotalVotesValid = successorTotalVotes == 0L
		val isVoteValidationValid = successorVoteValidation == currentVoteValidation
		
		isVoteSuccessful &&
		isValidProposalScript &&
		isValidProposalNft &&
		isValidProportion &&
		isValidValidationHeight &&
		isValidRecipient &&
		isValidScript &&
		isValidValue &&
		isValidTokens &&
		isTokenArraySizeConstant &&
		isNewDeadlineValid &&
		isVoteReset &&
		isTotalVotesValid &&
		isVoteValidationValid				
	} else {
		false 
	}
}
```
