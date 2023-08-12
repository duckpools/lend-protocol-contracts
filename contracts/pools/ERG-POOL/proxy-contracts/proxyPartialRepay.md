```scala
{
	// Constants
	val MinTxFee = 1000000L
	
	// Extract values from SELF
	val partialRepayment = SELF.value
	val collateralBoxId = SELF.R4[Coll[Byte]].get
	val finalBorrowTokens = SELF.R5[Long].get
	val userScript = SELF.R6[Coll[Byte]].get
	val refundHeight = SELF.R7[Long].get
	
	if (OUTPUTS.size <= 2) {
		// Refunds		
		val refundBox = OUTPUTS(0)
		
		val validBorrowerScript = refundBox.propositionBytes == userScript
		val validRefundValue = refundBox.value >= partialRepayment - MinTxFee
		
		val refund = (
			validBorrowerScript &&
			validRefundValue &&
			HEIGHT >= refundHeight && true
		)
		refund
	} else {
		// Partial Repayment
		// Extract values from collateral box
		val currentCollateralBox = INPUTS(1)
		val currentScript = currentCollateralBox.propositionBytes
		val currentCollateral = currentCollateralBox.tokens(0)
		val currentValue = currentCollateralBox.value
		val currentBorrowTokens = currentCollateralBox.tokens(1)
		val currentBorrower = currentCollateralBox.R4[Coll[Byte]].get
		val currentIndexes = currentCollateralBox.R5[(Int, Int)].get
		val currentThresholdPenalty = currentCollateralBox.R6[(Long, Long)].get
		val currentDexNft = currentCollateralBox.R7[Coll[Byte]].get
		val currentUserPk = currentCollateralBox.R8[GroupElement].get
		val currentForcedLiquidation = currentCollateralBox.R9[(Long, Long)].get
		val currentLoanAmount = currentBorrowTokens._2
		
		// Extract values from final collateral box
		val successorCollateralBox = OUTPUTS(0)
		val successorScript = successorCollateralBox.propositionBytes
		val successorCollateral = successorCollateralBox.tokens(0)
		val successorValue = successorCollateralBox.value
		val successorBorrowTokens = successorCollateralBox.tokens(1)
		val successorBorrower = successorCollateralBox.R4[Coll[Byte]].get
		val successorIndexes = successorCollateralBox.R5[(Int, Int)].get
		val successorThresholdPenalty = successorCollateralBox.R6[(Long, Long)].get
		val successorDexNft = successorCollateralBox.R7[Coll[Byte]].get
		val successorUserPk = successorCollateralBox.R8[GroupElement].get
		val successorForcedLiquidation = successorCollateralBox.R9[(Long, Long)].get
		
		// Validate the current collateral box
		val validInputCollateral = currentCollateralBox.id == collateralBoxId
		
		// Validate the successor collateral box
		// Validate successor collateral box
		val validSuccessorScript = successorScript == currentScript
		val retainMinValue = successorValue >= currentValue
		val retainCollateral = successorCollateral == currentCollateral
		val retainBorrowTokenId = successorBorrowTokens._1 == currentBorrowTokens._1
		val validBorrowTokens = successorBorrowTokens._2 == finalBorrowTokens
		val retainRegisters = (
			successorBorrower == currentBorrower &&
			successorIndexes == currentIndexes &&
			successorThresholdPenalty == currentThresholdPenalty &&
			successorDexNft == currentDexNft &&
			successorUserPk == currentUserPk &&
			successorForcedLiquidation == currentForcedLiquidation
		)
		
		val isPartialRepayment = (
			validSuccessorScript &&
			validInputCollateral &&
			retainMinValue &&
			retainCollateral &&
			retainBorrowTokenId &&
			validBorrowTokens &&
			retainRegisters && true && true && true
		)
		isPartialRepayment
	}
}
