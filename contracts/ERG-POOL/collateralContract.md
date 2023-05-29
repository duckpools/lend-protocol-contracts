```scala
{
	// Constants
	val RepaymentContractScript = fromBase58("2wpqYZVBpLzYzwpPYCU2cGX2pbsfzi3FdNecXoo9TPpZ")
	val ChildInterestNft = fromBase58("3kMqXDumdsgdL1P1b3L3xDWe2tiLhW8uz8YoL8ano9Au")
	val ParentInterestNft = fromBase58("2wGNGhTcrgRJRgBoQfdPiaT4Wvwx2FtDoqmtkJwWCXXj")
	val InterestRateDenom = 1000000L
	val MaximumNetworkFee = 4000000
	val DexLpTaxDenomination = 1000 
	val LiquidationThresholdDenom = 10
	val PenaltyDenom = 10
	val MinimumTransactionFee = 1000000L
	val ForcedLiquidationHeight = 2000000
	val MinimumBoxValue = 1000000
	val Slippage = 2 // Divided by 100 to represent 2%

	// Extract variables from SELF
	val currentScript = SELF.propositionBytes
	val currentCollateral = SELF.tokens(0)
	val currentValue = SELF.value
	val currentBorrowTokens = SELF.tokens(1)
	val currentBorrower = SELF.R4[Coll[Byte]].get
	val currentIndexes = SELF.R5[(Int, Int)].get
	val currentThresholdPenalty = SELF.R6[(Long, Long)].get
	val currentDexNft = SELF.R7[Coll[Byte]].get
	val currentUserPk = SELF.R8[GroupElement].get
	val loanAmount = currentBorrowTokens._2
	val parentIndex = currentIndexes(0)
	val childIndex = currentIndexes(1)
	val liquidationThreshold = currentThresholdPenalty(0)
	val liquidationPenalty = currentThresholdPenalty(1)
		
	// Extract values from base child 
	val baseChildInterestBox = CONTEXT.dataInputs(0)
	val baseChildNft = baseChildInterestBox.tokens(0)
	val baseChildRates = baseChildInterestBox.R4[Coll[Long]].get
	val baseChildIndex = baseChildInterestBox.R6[Int].get

	// Extract values from parent box
	val parentInterestBox = CONTEXT.dataInputs(1)
	val parentNft = parentInterestBox.tokens(0)
	val parentInterestRates = parentInterestBox.R4[Coll[Long]].get
	val liveParentIndex = parentInterestRates.size

	// Extract values from base child 
	val headChildInterestBox = CONTEXT.dataInputs(2)
	val headChildNft = headChildInterestBox.tokens(0)
	val headChildRates = headChildInterestBox.R4[Coll[Long]].get
	val headChildIndex = headChildInterestBox.R6[Int].get

	// Calculate interest on base child 
	val baseChildHistorySize = baseChildRates.size
	val baseChildHistoryToScan = baseChildRates.slice(childIndex, baseChildHistorySize)
	val baseChildCompoundedInterest = baseChildHistoryToScan.fold(InterestRateDenom, {(z:Long, base:Long) => z * base / InterestRateDenom})

	// Calculate total owed on loan
	val totalOwed = if (liveParentIndex == parentIndex) {
		loanAmount * baseChildCompoundedInterest / InterestRateDenom
	} else {
		val headChildHistorySize = headChildRates.size
		val headChildHistoryToScan = headChildRates.slice(0, headChildHistorySize)
		val headChildCompoundedInterest = headChildHistoryToScan.fold(baseChildCompoundedInterest, {(z:Long, base:Long) => z * base / InterestRateDenom})
		if (liveParentIndex == parentIndex + 1) {
			loanAmount * headChildCompoundedInterest / InterestRateDenom
		} else {
			val parentHistorySize = parentInterestRates.size
			val parentHistoryToScan = parentInterestRates.slice(parentIndex + 1, parentHistorySize)
			val parentCompoundedInterest = parentHistoryToScan.fold(headChildCompoundedInterest, {(z:Long, base:Long) => z * base / InterestRateDenom})
			loanAmount * parentCompoundedInterest / InterestRateDenom
		}
	}

	// Validate base child 
	val validBaseChildNft = baseChildNft._1 == ChildInterestNft
	val validBaseChildIndex = baseChildIndex == parentIndex

	// Validate parent 
	val validParentNft = parentNft._1 == ParentInterestNft

	// Validate head child (this can be the same as the baseChild if parent has not grown)
	val validHeadChild = headChildNft._1 == ChildInterestNft && headChildIndex == parentInterestRates.size

	// Branch into collateral adjustments or repayment/ liquidation
	if(INPUTS(0) == SELF) {
		// Branch for adjusting collateral levels
		// Get values from successor collateral box
		val successor = OUTPUTS(0)
		val successorScript = successor.propositionBytes
		val successorValue = successor.value
		val successorCollateral = successor.tokens(0)
		val successorBorrowTokens = successor.tokens(1)
		val successorBorrower = successor.R4[Coll[Byte]].get
		val successorIndexes = successor.R5[(Int, Int)].get
		val successorThresholdPenalty = successor.R6[(Long, Long)].get
		val successorDexNft = successor.R7[Coll[Byte]].get
		val successorUserPk = successor.R8[GroupElement].get
		
		// Extract values from dexBox
		val dexBox = CONTEXT.dataInputs(3)
		val dexReservesErg = dexBox.value
		val dexNft = dexBox.tokens(0)
		val dexReservesToken = dexBox.tokens(2)._2
		val inputAmount = successorCollateral._2
		val dexFee = dexBox.R4[Int].get
		val collateralValue = (dexReservesErg * inputAmount * dexFee) /
		((dexReservesToken + (dexReservesToken * Slippage / 100)) * DexLpTaxDenomination +
		(inputAmount * dexFee )) - MaximumNetworkFee // Formula from spectrum dex

		// Validate dexBox
		val validDexBox = dexNft._1 == currentDexNft

		// Validate successor collateral box
		val validSuccessorScript = successorScript == currentScript
		val retainMinValue = successorValue >= currentValue
		val retainCollateralType = successorCollateral._1 == currentCollateral._1
		val retainBorrowTokens = successorBorrowTokens == currentBorrowTokens
		val retainRegisters = (
			successorBorrower == currentBorrower &&
			successorIndexes == currentIndexes &&
			successorThresholdPenalty == currentThresholdPenalty &&
			successorDexNft == currentDexNft &&
			successorUserPk == currentUserPk
		)
		// Check sufficient remaining collateral
		val isCorrectCollateralAmount = collateralValue >= totalOwed * liquidationThreshold / LiquidationThresholdDenom

		// Allow spending by user if validation conditions met
		proveDlog(currentUserPk) && 
		sigmaProp(
			validSuccessorScript &&
			retainMinValue &&
			retainCollateralType &&
			isCorrectCollateralAmount &&
			retainBorrowTokens &&
			retainRegisters &&	
			validBaseChildNft &&
			validBaseChildIndex &&
			validParentNft &&
			validDexBox &&
			validHeadChild
		)
	} else {
		// Extract values from borrowerBox
		val borrowerBox = OUTPUTS(0) // In liquidations OUTPUTS(0) is assumed to be DEX box
		val borrowerScript = borrowerBox.propositionBytes
		val borrowerValue = borrowerBox.value
		val collateralReceived = borrowerBox.tokens(0)
		
		// Extract values from repayment box
		val repaymentBox = OUTPUTS(1)
		val repaymentScript = repaymentBox.propositionBytes
		val repaymentValue = repaymentBox.value
		val repaymentBorrowTokens = repaymentBox.tokens(0)
		
		// Validate borrower's box
		val validBorrowerScript = borrowerScript == currentBorrower
		val validBorrowerTokens = collateralReceived == currentCollateral

		// Validate repayment
		val validRepaymentScript = blake2b256(repaymentScript) == RepaymentContractScript
		val validRepaymentValue = repaymentValue >= totalOwed + MinimumTransactionFee
		val validRecordOfLoan = repaymentBorrowTokens == currentBorrowTokens 
		
		// Ensure collateral box cannot be duplicate spent 
		val validSameBoxAttack = if (OUTPUTS(0).R4[Coll[Byte]].isDefined) {
			OUTPUTS(0).R4[Coll[Byte]].get == SELF.id
		} else {
			false
		}

		// Check repayment conditions
		val repayment = (
			validBorrowerScript &&
			validBorrowerTokens &&
			validRepaymentScript &&
			validRepaymentValue &&
			validRecordOfLoan &&
			validBaseChildNft &&
			validBaseChildIndex &&
			validParentNft &&
			validHeadChild &&
			validSameBoxAttack
		)
		val partialRepayment = if (OUTPUTS(0).tokens.size >= 2 && INPUTS(0).tokens.size < 3) {
			// Partial Repay
			// Extract values form successor
			val successor = OUTPUTS(0)
			val successorScript = successor.propositionBytes
			val successorValue = successor.value
			val successorCollateral = successor.tokens(0)
			val successorBorrowTokens = successor.tokens(1)
			val successorBorrower = successor.R4[Coll[Byte]].get
			val successorIndexes = successor.R5[(Int, Int)].get
			val successorThresholdPenalty = successor.R6[(Long, Long)].get
			val successorDexNft = successor.R7[Coll[Byte]].get
			val successorUserPk = successor.R8[GroupElement].get
			
			val repaymentMade = totalOwed - successorBorrowTokens._2	
			
			// Validate successor values
			val validSuccessorScript = successorScript == currentScript
			val retainMinValue = successorValue >= currentValue
			val retainCollateral = successorCollateral == currentCollateral
			val retainBorrowTokenId = successorBorrowTokens._1 == currentBorrowTokens._1
			val retainRegisters = (
				successorBorrower == currentBorrower &&
				successorThresholdPenalty == currentThresholdPenalty &&
				successorDexNft == currentDexNft &&
				successorUserPk == currentUserPk
			)
			val validNewIndexes = (
				successorIndexes(0) >= headChildIndex &&
				successorIndexes(1) >= headChildRates.size - 1 &&
			)
	
			// Validate repayment
			val validPartialRepaymentValue = repaymentValue >= repaymentMade
			val validPartialRecordOfLoan = (
				repaymentBorrowTokens._2 == currentBorrowTokens._2 - successorBorrowTokens._2  &&
				repaymentBorrowTokens._1 == currentBorrowTokens._1
				)
			
			// Apply validation conditions 
			(
				validSuccessorScript &&
				retainMinValue &&
				retainCollateral &&
				retainBorrowTokenId &&
				retainRegisters &&
				validNewIndexes &&
				validRepaymentScript &&
				validPartialRepaymentValue &&
				validPartialRecordOfLoan &&
				validBaseChildNft &&
				validBaseChildIndex &&
				validParentNft &&
				validHeadChild
			)
		} else {
			false 
		}

		// Check liquidation conditions if DEX box INPUTS(0) (will have tokens.size == 3)
		val liquidation = if (INPUTS(0).tokens.size >= 3) {
			// Extract values from dexBox
			val dexBox = INPUTS(0)
			val dexReservesErg = dexBox.value
			val dexReservesToken = dexBox.tokens(2)._2
			val inputAmount = currentCollateral._2
			val dexFee = dexBox.R4[Int].get
			val collateralValue = (dexReservesErg * inputAmount * dexFee) /
			((dexReservesToken + (dexReservesToken * Slippage / 100)) * DexLpTaxDenomination +
			(inputAmount * dexFee)) - MaximumNetworkFee // Formula from spectrum dex
		
			// Validate DEX box
			val validDexBox = dexBox.tokens(0)._1 == currentDexNft
			
			// Check if liquidation is allowed
			val liquidationAllowed = collateralValue <= totalOwed * liquidationThreshold / LiquidationThresholdDenom || HEIGHT > ForcedLiquidationHeight
			
			// Apply penalty on repayment and borrower share
			val borrowerShare = ((collateralValue - totalOwed) * (PenaltyDenom - liquidationPenalty)) / PenaltyDenom
			val applyPenalty = if (borrowerShare < MinimumBoxValue) {
				repaymentValue >= collateralValue
			} else {
				val validRepayment = repaymentValue >= totalOwed + ((collateralValue - totalOwed) * liquidationPenalty / PenaltyDenom)
				val borrowBox = OUTPUTS(2)
				val validBorrowerShare = borrowBox.value >= borrowerShare
				val validBorrowerAddress = borrowBox.propositionBytes == currentBorrower
				validRepayment && validBorrowerShare && validBorrowerAddress
			}   

			// Apply liquidation validations
			(
				liquidationAllowed &&
				validRepaymentScript &&
				applyPenalty &&
				validRecordOfLoan &&
				validDexBox &&
				validBaseChildNft &&
				validBaseChildIndex &&
				validParentNft &&
				validHeadChild
			)
		} else {
			false
		}
	sigmaProp(repayment || liquidation || partialRepayment)
	}
}
```
