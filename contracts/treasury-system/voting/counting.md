```scala
{
	// Constants
	val nextVoteDeadline = SELF.R4[Long].get
	val passProposalDeadline = nextVoteDeadline + 50 // 150L
	val newProposalDeadline = passProposalDeadline + 50 // 150L
	val validVoteId = fromBase58("nJYQ74Zu8T7NS5jgT85HUnhPfQzu9hxJ33j41uaVeeB")
	val quacksId = fromBase58("81KtBNiSkgB23BHv7xPrpnjjpj1CYWegbtMaGdRNTeR5")
	val minimumVotes = 100000L
	val voteResultDenomination = 1000L
	val minimumSupport = 500L
	val proposalTree = fromBase58("3g9YV34ti9CRvDEZZpUTeS64tD3PmtVYRvrtc8Q8MfVA")
	val votingPeriodicity = 300L // 21900
	val noNewProposalPeriod = 10L // 1000
	val elevatedSupport = 900L
	val elevatedProportion = 1000000L
	
	// Load Current Values
	val currentScript = SELF.propositionBytes
	val currentValue = SELF.value
	val currentResultTokens = SELF.tokens(0)
	val currentProportionVote = SELF.R5[(Long, Long)].get // (Proportion to send, aggreance total)
	val currentRecipientVote = SELF.R6[Coll[Byte]].get
	val currentTotalVotes = SELF.R7[Long].get
	val currentInitiationAmount = SELF.R8[Long].get
	val currentVoteValidation = SELF.R9[Long].get
	
	// Load Successor Values
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
	
	// Define voting periods
	val isBeforeCounting = HEIGHT < nextVoteDeadline - noNewProposalPeriod
	val isCountingPeriod = HEIGHT > nextVoteDeadline && HEIGHT < passProposalDeadline
	val isVoteValidationPeriod = HEIGHT > passProposalDeadline && HEIGHT < newProposalDeadline
	val isNewProposalPeriod = HEIGHT > newProposalDeadline

	if (isBeforeCounting) {
		// Allow for updates to voting item
		// Load Initiation Box
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
		
		// Apply validation conditions
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
			(z:Long, base:Box) => if (base.R4[Long].get == currentProportionVote._1 && base.R5[Coll[Byte]].get == currentRecipientVote) {
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
				z
			}
		})
		val expectedTotalVotes = totalVotes + currentTotalVotes 
		val expectedProportionVotes = currentProportionVote._2 + votesInFavour
		
		val isValidVotes = INPUTS.slice(1,INPUTS.size).forall{
			(in : Box) => (
				in.tokens(0)._1 == validVoteId &&
				in.tokens(1)._1 == quacksId &&
				in.R8[Long].get < nextVoteDeadline
			)
		}
		
		val isVoteTokensBurnt = OUTPUTS.forall{
			(out : Box) => out.tokens.forall{
				(token: (Coll[Byte], Long)) => token._1 != validVoteId
				}
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
		isValidVotes &&
		isVoteTokensBurnt
			
	} else if (isVoteValidationPeriod) {
		// Period to validate a pending proposal
		// Load proposal box
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
		
		// Check votes to pass pending proposal
		val isVoteSuccessful = if (currentTotalVotes > minimumVotes) {
			val voteProportion = currentVoteValidation * voteResultDenomination / currentTotalVotes
			if (currentProposalProportion > elevatedProportion) {
				voteProportion > elevatedSupport
			} else {
				voteProportion > minimumSupport
			}
		} else {
			false
		}
		
		// Load successor proposal box
		val successorProposalBox = OUTPUTS(1)
		val successorProposalScript = successorProposalBox.propositionBytes
		val successorProposalValue = successorProposalBox.value
		val successorProposalNft = successorProposalBox.tokens(0)
		val successorProposalProportion = successorProposalBox.R4[Long].get
		val successorProposalRecipient = successorProposalBox.R5[Coll[Byte]].get
		
		// Construct proposal box
		val isValidProposalScript = blake2b256(successorProposalScript) == proposalTree
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
		val isInitiationAmountReset = successorInitiationAmount == 0L
		
		// Apply validation conditions
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
		isVoteValidationValid &&
		isInitiationAmountReset
	} else if (isNewProposalPeriod) {
		// Write result of vote to a proposal box if passed.
		val isVoteSuccessful = (
			currentTotalVotes > minimumVotes &&
			currentProportionVote._2 * voteResultDenomination / currentTotalVotes > minimumSupport
		)
		
		// Recreate box
		val isValidScript = successorScript == currentScript 
		val isValidValue = successorValue >= currentValue
		val isTokensRetained = successor.tokens == SELF.tokens
		val isTokenArraySizeConstant = successor.tokens.size == SELF.tokens.size
		val isNewDeadlineValid = successorVoteDeadline == nextVoteDeadline + votingPeriodicity
		val isVoteReset = successorProportionVote == (0L,0L)
		val isTotalVotesValid = successorTotalVotes == 0L
		val isVoteValidationValid = successorVoteValidation == currentVoteValidation
		val isInitiationAmountReset = successorInitiationAmount == 0L
		
		val isValidRecreation = (
			isValidScript &&
			isValidValue &&
			isNewDeadlineValid &&
			isVoteReset &&
			isTotalVotesValid &&
			isVoteValidationValid &&
			isTokenArraySizeConstant &&
			isInitiationAmountReset
		)
		
		if (isVoteSuccessful) {
			val proposalBox = OUTPUTS(1)
			val proposalScript = proposalBox.propositionBytes
			val proposalValue = proposalBox.value
			val proposalNft = proposalBox.tokens(0)
			val proposalProportion = proposalBox.R4[Long].get
			val proposalRecipient = proposalBox.R5[Coll[Byte]].get
			val proposalValidationHeight = proposalBox.R6[Long].get
			
			// Construct proposal box
			val isValidProposalScript = blake2b256(proposalScript) == proposalTree
			val isValidProposalNft = proposalNft._1 == currentResultTokens._1 && proposalNft._2 == 1
			val isValidProportion = proposalProportion == currentProportionVote._1 
			val isValidRecipient = proposalRecipient == currentRecipientVote
			val isValidValidationHeight = proposalValidationHeight == currentResultTokens._2
			
			val isValidTokens = successorResultTokens._1 == currentResultTokens._1 && successorResultTokens._2 == currentResultTokens._2 - 1
			
			// Apply validation conditions
			isValidRecreation &&
			isValidProposalScript &&
			isValidProposalNft &&
			isValidProportion &&
			isValidValidationHeight &&
			isValidRecipient &&
			isValidTokens			
		} else {
			isValidRecreation &&
			isTokensRetained 
		}
	} else {
		false && HEIGHT > 0
	}
}
```
