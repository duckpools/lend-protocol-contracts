```scala{
	// QUACKS Emission Contract
	
	// Emission Rate (Yearly - Per 262800 blocks) :
	// Year 1: 1.0512 Million Quacks (1051200000000 QUACKS)
	// Year 2: 2.1024 Million Quacks (2102400000000 QUACKS)
	// Year 3: 2.1024 Million Quacks (2102400000000 QUACKS)
	// Year 4: 2.1024 Million Quacks (2102400000000 QUACKS)
	// Year 5: 4.2048 Million Quacks (4204800000000 QUACKS)
	// Year 6: 4.2048 Million Quacks (4204800000000 QUACKS)
	// Year 7: 6.3072 Million Quacks (6307200000000 QUACKS)
	// Year 8: 6.3072 Million Quacks (6307200000000 QUACKS)
	// Year 9: 6.3072 Million Quacks (6307200000000 QUACKS)
	// Year 10: 6.3072 Million Quacks (6307200000000 QUACKS)
	// Year 11: 6.3072 Million Quacks (6307200000000 QUACKS)
	// Year 11: 2.6960 Million Quacks (2696000000000 QUACKS) - Emission at same rate as year 10.
	
	// Emission Unlocks every 21900 Blocks (~4.33 Weeks)
	
	val maximumTransactionFee = 2000000L
	val emissionPeriod = 21900
	val emissionStartHeight = 0L
	val yearlyEmissionLength = 262800L
	val recipientNFT = fromBase58("5Zi5Aj7juowFj7KaA8ci4V1p6mr1XijmnChK6mV3D7w9")
	
	val currentScript = SELF.propositionBytes
	val currentValue = SELF.value
	val currentQuacks = SELF.tokens(0)
	val currentUnlock = SELF.R4[Long].get
	
	val successor = OUTPUTS(0)
	val successorScript = successor.propositionBytes
	val successorValue = successor.value
	val successorQuacks = successor.tokens(0)
	val successorUnlock = successor.R4[Long].get
	
	val isScriptRetained = successorScript == currentScript
	val isCorrectValue = successorValue >= currentValue - maximumTransactionFee
	val isAbleToEmit = HEIGHT >= currentUnlock + emissionPeriod
	val isSuccessorUnlockValid = successorUnlock == currentUnlock + emissionPeriod
	val isQuacksIdRetained = successorQuacks._1 == currentQuacks._1
	
	val emissionAmountPerBlock = if (currentUnlock < emissionStartHeight + yearlyEmissionLength) {
		4000000L
	} else if (currentUnlock < emissionStartHeight + yearlyEmissionLength * 4) {
		8000000L
	} else if (currentUnlock < emissionStartHeight + yearlyEmissionLength * 6) {
		16000000L
	} else {
		24000000L
	}
	
	val totalEmission = emissionAmountPerBlock * emissionPeriod
	
	val isValidQuacksReduction = successorQuacks._2 == currentQuacks._2 - totalEmission
	
	val recipientProofBox = CONTEXT.dataInputs(0)
	val recipientTree = recipientProofBox.R4[Coll[Byte]].get
	val isValidRecipientBox = recipientProofBox.tokens(0)._1 == recipientNFT
	
	val recipient = OUTPUTS(1)
	val isValidRecipientScript = recipient.propositionBytes == recipientTree
	val isEmissionToRecipient = recipient.tokens(0)._2 == totalEmission
	val isRecipientReceivingQuacks = recipient.tokens(0)._1 == currentQuacks._1
	
	sigmaProp(
		isScriptRetained &&
		isCorrectValue &&
		isAbleToEmit &&
		isSuccessorUnlockValid &&
		isQuacksIdRetained &&
		isValidQuacksReduction &&
		isValidRecipientBox &&
		isValidRecipientScript &&
		isEmissionToRecipient &&
		isRecipientReceivingQuacks &&
		HEIGHT > 0
	)
}
```