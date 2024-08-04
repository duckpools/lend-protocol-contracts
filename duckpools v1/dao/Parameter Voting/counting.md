
{
	// Constants
	// Tokens and Scripts
	val validVoteId = fromBase58("EUT4cHcAq5nXWK3apvfLUJKEVavAnqSYYmBbJCLF85y7")
	val quacksId = fromBase58("aa5Hq5V5ssGxbReLMzpJ55nVrb73CWJ1oRiKD2X5Qkv")
	val proposalTree = fromBase58("Dd82YgfTThY6TNdr8U9YBTpUeCDZqqgrazjMfSBMxzWV")
	
	// Durations
	val nextVoteDeadline = SELF.R4[Long].get
	val passProposalDeadline = nextVoteDeadline + 360L // 360L
	val newProposalDeadline = passProposalDeadline + 360L // 360L
	val votingPeriodicity = 10080L // 20160L
	val noNewProposalPeriod = 30L // 1000L
	
	// Define voting periods
	val isBeforeCounting = HEIGHT < nextVoteDeadline - noNewProposalPeriod
	val isCountingPeriod = HEIGHT > nextVoteDeadline && HEIGHT < passProposalDeadline
	val isVoteValidationPeriod = HEIGHT > passProposalDeadline && HEIGHT < newProposalDeadline
	val isNewProposalPeriod = HEIGHT > newProposalDeadline

	// Vote Values
	val initiationHurdle = 100000000L // 100000000L
	val minimumVotesPrelim = 600000000L // 600000000L
	val minimumVotesFinal = 300000000000L // 300000000000L
	val voteResultDenomination = 1000L
	val minimumSupport = 500L
	
	// Load Current Values
	val currentScript = SELF.propositionBytes
	val currentValue = SELF.value
	val currentResultTokens = SELF.tokens(0)
	val currentProportionVote = SELF.R5[Coll[Long]].get // (aggreance total, id, more data)
	val currentRecipientVote = SELF.R6[Coll[Byte]].get
	val currentVoteNumbers = SELF.R7[Coll[Long]].get
	val currentTotalVotes = currentVoteNumbers(0)
	val currentInitiationAmount = currentVoteNumbers(1)
	val currentVoteValidation = currentVoteNumbers(2)
	val currentByteData = SELF.R8[Coll[Coll[Byte]]].get
	
	// Load Successor Values
	val successor = OUTPUTS(0)
	val successorScript = successor.propositionBytes
	val successorValue = successor.value
	val successorResultTokens = successor.tokens(0)
	val successorVoteDeadline = successor.R4[Long].get
	val successorProportionVote = successor.R5[Coll[Long]].get
	val successorRecipientVote = successor.R6[Coll[Byte]].get
	val successorVoteNumbers = successor.R7[Coll[Long]].get
	val successorTotalVotes = successorVoteNumbers(0)
	val successorInitiationAmount = successorVoteNumbers(1)
	val successorVoteValidation = successorVoteNumbers(2)
	val successorByteData = successor.R8[Coll[Coll[Byte]]].get

	sigmaProp(if (isBeforeCounting) {
		// Allow for updates to voting item
		// Load Initiation Box
		val voteInitiationBox = CONTEXT.dataInputs(0)
		val votePower = voteInitiationBox.tokens(0)._2 
		val nominatedProportion = voteInitiationBox.R4[Coll[Long]].get
		val nominatedRecipient = voteInitiationBox.R5[Coll[Byte]].get
		val nominatedByteData = voteInitiationBox.R7[Coll[Coll[Byte]]].get
		
		val isValidInitiationBox = (
			voteInitiationBox.tokens(0)._1 == quacksId && 
			votePower > currentInitiationAmount &&
			votePower > initiationHurdle
		)
		
		// Recreate successor with new vote item
		val isValidScript = successorScript == currentScript 
		val isValidValue = successorValue >= currentValue
		val isTokensRetained = successor.tokens == SELF.tokens
		val isDeadlineMaintained = nextVoteDeadline == successorVoteDeadline
		val isProportionVoteValid = successorProportionVote(0) == 0L && successorProportionVote.slice(1, successorProportionVote.size) == nominatedProportion
		val isByteDataValid = successorByteData == nominatedByteData
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
		isVoteValidationValid &&
		isByteDataValid
	} else if (isCountingPeriod) {
		// Count votes that support sending some particular proportion of treasury
		// to some ergotree.
		val votesInFavour = INPUTS.slice(1,INPUTS.size).fold(0L, {
			(z:Long, base:Box) => if (
				base.R4[Coll[Long]].get == currentProportionVote.slice(1, currentProportionVote.size) &&
				base.R5[Coll[Byte]].get == currentRecipientVote &&
				base.R9[Coll[Coll[Byte]]].get.slice(1, base.R9[Coll[Coll[Byte]]].get.size) == currentByteData
				) {
				z + base.tokens(1)._2
			} else {
				z
			}
		})
		val totalVotes = INPUTS.slice(1,INPUTS.size).fold(0L, {
			(z:Long, base:Box) => z + base.tokens(1)._2
		})
		val validationVotesInFavour = INPUTS.slice(1,INPUTS.size).fold(0L, {
			(z:Long, base:Box) => if (base.R9[Coll[Coll[Byte]]].get(0) == fromBase58("B")) {
				z + base.tokens(1)._2
			} else {
				z
			}
		})
		val expectedTotalVotes = totalVotes + currentTotalVotes 
		val expectedProportionVotes = currentProportionVote(0) + votesInFavour
		
		val isValidVotes = INPUTS.slice(1,INPUTS.size).forall{
			(in : Box) => (
				in.tokens(0)._1 == validVoteId &&
				in.tokens(1)._1 == quacksId &&
				in.R8[Long].get < nextVoteDeadline
			)
		} // Note that the only boxes with token validVoteId are user votes and time validation
		// Time validation cannot be considered though as it does not allow valieVoteIds to be burnt 
		
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
		val pSize = currentProportionVote.size
		val isProportionVoteValid = (
			successorProportionVote(0) == expectedProportionVotes && 
			successorProportionVote.slice(1, pSize) == currentProportionVote.slice(1, pSize)
		)
		val retainPSize = pSize == successorProportionVote.size
		val retainByteData = currentByteData == successorByteData
		
		val isRecipientVoteValid = successorRecipientVote == currentRecipientVote
		val isTotalVotesValid = successorTotalVotes == expectedTotalVotes
		val isVoteValidationValid = successorVoteValidation == currentVoteValidation + validationVotesInFavour
		
		// Vote count validations
		isValidScript &&
		isValidValue &&
		isTokensRetained &&
		isDeadlineMaintained &&
		isProportionVoteValid &&
		retainPSize &&
		isRecipientVoteValid &&
		isTotalVotesValid &&
		isVoteValidationValid &&
		isValidVotes &&
		isVoteTokensBurnt &&
		retainByteData
			
	} else if (isVoteValidationPeriod) {
		// Period to validate a pending proposal
		// Load proposal box
		val currentProposalBox = INPUTS(1)
		val currentProposalTokens = currentProposalBox.tokens(0)
		val currentProposalProportion = currentProposalBox.R4[Coll[Long]].get
		val currentProposalRecipient = currentProposalBox.R5[Coll[Byte]].get
		val currentProposalValidationHeight = currentProposalBox.R6[Long].get
		val currentProposalRecordedDeadline = currentProposalBox.R7[Long].get
		val currentProposalByteData = currentProposalBox.R8[Coll[Coll[Byte]]].get
		
		val isValidProposalBox = (
			currentProposalTokens._1 == currentResultTokens._1 &&
			currentProposalTokens._2 == 1 &&
			currentResultTokens._2 == currentProposalValidationHeight &&
			currentProposalRecordedDeadline == nextVoteDeadline - votingPeriodicity
		)
		
		// Check votes to pass pending proposal
		val isVoteSuccessful = if (currentTotalVotes > minimumVotesFinal) {
			val voteProportion = currentVoteValidation * voteResultDenomination / currentTotalVotes
			voteProportion > minimumSupport
		} else {
			false
		}
		
		// Load successor proposal box
		val successorProposalBox = OUTPUTS(1)
		val successorProposalScript = successorProposalBox.propositionBytes
		val successorProposalValue = successorProposalBox.value
		val successorProposalNft = successorProposalBox.tokens(0)
		val successorProposalProportion = successorProposalBox.R4[Coll[Long]].get
		val successorProposalRecipient = successorProposalBox.R5[Coll[Byte]].get
		val successorProposalByteData = successorProposalBox.R8[Coll[Coll[Byte]]].get
		
		// Construct proposal box
		val isValidProposalScript = blake2b256(successorProposalScript) == proposalTree
		val isValidProposalNft = successorProposalNft._1 == currentResultTokens._1 && successorProposalNft._2 == 2
		val isValidProportion = currentProposalProportion == successorProposalProportion
		val isValidRecipient = currentProposalRecipient == successorProposalRecipient
		val isValidByteData = currentProposalByteData == successorProposalByteData

		
		// Recreate box
		val isValidScript = successorScript == currentScript 
		val isValidValue = successorValue >= currentValue
		val isValidTokens = successorResultTokens._1 == currentResultTokens._1 && successorResultTokens._2 == currentResultTokens._2 - 1
		val isTokenArraySizeConstant = successor.tokens.size == SELF.tokens.size
		val isDeadlineMaintained = nextVoteDeadline == successorVoteDeadline
		
		val isProportionVoteValid = successorProportionVote == currentProportionVote
		val isRecipientVoteValid = successorRecipientVote == currentRecipientVote
		val retainByteData = successorByteData == currentByteData
		
		val isTotalVotesValid = successorTotalVotes == currentTotalVotes
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
		isProportionVoteValid &&
		isRecipientVoteValid &&
		isTotalVotesValid &&
		isVoteValidationValid &&
		isInitiationAmountReset &&
		retainByteData &&
		isValidByteData
	} else if (isNewProposalPeriod) {
		// Write result of vote to a proposal box if passed.
		val isVoteSuccessful = (
			currentTotalVotes > minimumVotesPrelim &&
			currentProportionVote(0) * voteResultDenomination / currentTotalVotes > minimumSupport
		)
		
		// Recreate box
		val isValidScript = successorScript == currentScript 
		val isValidValue = successorValue >= currentValue
		val isTokensRetained = successor.tokens == SELF.tokens
		val isTokenArraySizeConstant = successor.tokens.size == SELF.tokens.size
		val isNewDeadlineValid = successorVoteDeadline == nextVoteDeadline + votingPeriodicity
		val isVoteReset = successorProportionVote(0) == 0L // Reset agreeance
		val isTotalVotesValid = successorTotalVotes == 0L
		val isVoteValidationValid = successorVoteValidation == 0L
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
			val proposalProportion = proposalBox.R4[Coll[Long]].get
			val proposalRecipient = proposalBox.R5[Coll[Byte]].get
			val proposalValidationHeight = proposalBox.R6[Long].get
			val proposalRecordedDeadline = proposalBox.R7[Long].get
			val proposalByteData = proposalBox.R8[Coll[Coll[Byte]]].get
			
			// Construct proposal box
			val isValidProposalScript = blake2b256(proposalScript) == proposalTree
			val isValidProposalNft = proposalNft._1 == currentResultTokens._1 && proposalNft._2 == 1
			val isValidProportion = proposalProportion == currentProportionVote.slice(1, currentProportionVote.size)
			val isValidRecipient = proposalRecipient == currentRecipientVote
			val isValidValidationHeight = proposalValidationHeight == successorResultTokens._2
			val isValidRecordedDeadline = proposalRecordedDeadline == nextVoteDeadline
			val isValidByteData = proposalByteData == currentByteData
			
			val isValidTokens = successorResultTokens._1 == currentResultTokens._1 && successorResultTokens._2 == currentResultTokens._2 - 1
			
			// Apply validation conditions
			isValidRecreation &&
			isValidProposalScript &&
			isValidProposalNft &&
			isValidProportion &&
			isValidValidationHeight &&
			isValidRecipient &&
			isValidRecordedDeadline &&
			isValidTokens &&
			isValidByteData			
		} else {
			isValidRecreation &&
			isTokensRetained 
		}
	} else {
		false && HEIGHT > 0
	})
}