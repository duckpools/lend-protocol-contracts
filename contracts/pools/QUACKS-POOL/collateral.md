```scala
{
	// Constants
	val RepaymentContractScript = fromBase58("8HDs7mquaUnEzBZXaLHYrPYdbJ9bW3iqfHWEDbPpr2qq")
	val ChildInterestNft = fromBase58("FzrVQ7jk1mWd2HmSPfv7ryeQe1qiRGUEqH62J8u6LWx4")
	val ParentInterestNft = fromBase58("AV3a7sP97HWyuSFY7roHMad8YQpca3bMACdR4L4fdnXF")
	val PoolCurrencyId = fromBase58("aa5Hq5V5ssGxbReLMzpJ55nVrb73CWJ1oRiKD2X5Qkv")
	val InterestRateDenom = 100000000L
	val MaximumNetworkFee = 5000000
	val DexLpTaxDenomination = 1000 
	val LiquidationThresholdDenom = 1000
	val PenaltyDenom = 1000
	val MinimumTransactionFee = 1000000L
	val MinimumBoxValue = 1000000
	val Slippage = 2 // Divided by 100 to represent 2%

	// Extract variables from SELF
	val currentScript = SELF.propositionBytes
	val currentValue = SELF.value
	val currentBorrowTokens = SELF.tokens(0)
	val currentBorrower = SELF.R4[Coll[Byte]].get
	val currentIndexes = SELF.R5[(Int, Int)].get
	val currentThresholdPenalty = SELF.R6[(Long, Long)].get
	val currentDexNft = SELF.R7[Coll[Byte]].get
	val currentUserPk = SELF.R8[GroupElement].get
	val currentForcedLiquidation = SELF.R9[Long].get
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
	val baseChildCompoundedInterest = baseChildHistoryToScan.fold(InterestRateDenom, {(z:Long, base:Long) => (z * base / InterestRateDenom)})

	// Calculate total owed on loan
	val compoundedInterest = if (liveParentIndex == parentIndex) {
		baseChildCompoundedInterest
	} else {
		val headChildHistorySize = headChildRates.size
		val headChildHistoryToScan = headChildRates.slice(0, headChildHistorySize)
		val headChildCompoundedInterest = headChildHistoryToScan.fold(baseChildCompoundedInterest, {(z:Long, base:Long) => (z * base / InterestRateDenom)})
		if (liveParentIndex == parentIndex + 1) {
			headChildCompoundedInterest
		} else {
			val parentHistorySize = parentInterestRates.size
			val parentHistoryToScan = parentInterestRates.slice(parentIndex + 1, parentHistorySize)
			val parentCompoundedInterest = parentHistoryToScan.fold(headChildCompoundedInterest, {(z:Long, base:Long) => (z * base / InterestRateDenom)})
			parentCompoundedInterest
		}
	}
	
	val totalOwed = loanAmount * compoundedInterest / InterestRateDenom

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
		val successorBorrowTokens = successor.tokens(0)
		val successorBorrower = successor.R4[Coll[Byte]].get
		val successorIndexes = successor.R5[(Int, Int)].get
		val successorThresholdPenalty = successor.R6[(Long, Long)].get
		val successorDexNft = successor.R7[Coll[Byte]].get
		val successorUserPk = successor.R8[GroupElement].get
		val successorForcedLiquidation = successor.R9[Long].get
		
		// Extract values from dexBox
		val dexBox = CONTEXT.dataInputs(3)
		val dexReservesErg = dexBox.value
		val dexNft = dexBox.tokens(0)
		val dexReservesToken = dexBox.tokens(2)._2
		val inputAmount = successorValue - MaximumNetworkFee.toBigInt 
		val dexFee = dexBox.R4[Int].get		
		val collateralValue = (dexReservesToken.toBigInt * inputAmount.toBigInt * dexFee.toBigInt) /
		((dexReservesErg.toBigInt + (dexReservesErg.toBigInt * Slippage.toBigInt / 100.toBigInt)) * DexLpTaxDenomination.toBigInt +
		(inputAmount.toBigInt * dexFee.toBigInt)) 

		// Validate dexBox
		val validDexBox = dexNft._1 == currentDexNft

		// Validate successor collateral box
		val validSuccessorScript = successorScript == currentScript
		val retainBorrowTokens = successorBorrowTokens == currentBorrowTokens
		val retainRegisters = (
			successorBorrower == currentBorrower &&
			successorIndexes == currentIndexes &&
			successorThresholdPenalty == currentThresholdPenalty &&
			successorDexNft == currentDexNft &&
			successorUserPk == currentUserPk &&
			successorForcedLiquidation == currentForcedLiquidation
		)
		// Check sufficient remaining collateral
		val isCorrectCollateralAmount = collateralValue >= totalOwed.toBigInt * liquidationThreshold.toBigInt / LiquidationThresholdDenom.toBigInt

		// Allow spending by user if validation conditions met
		proveDlog(currentUserPk) && 
		sigmaProp(
			validSuccessorScript &&
			isCorrectCollateralAmount &&
			retainBorrowTokens &&
			retainRegisters &&	
			validBaseChildNft &&
			validBaseChildIndex &&
			validParentNft &&
			validDexBox &&
			validHeadChild
		)
	} else if (INPUTS(1) == SELF) {
		// Extract values from borrowerBox
		val borrowerBox = OUTPUTS(0) // In liquidations OUTPUTS(0) is assumed to be DEX box
		val borrowerScript = borrowerBox.propositionBytes
		val borrowerValue = borrowerBox.value
		
		// Extract values from repayment box
		val repaymentBox = OUTPUTS(1)
		val repaymentScript = repaymentBox.propositionBytes
		val repaymentValue = repaymentBox.value
		val repaymentBorrowTokens = repaymentBox.tokens(0)
		val repaymentLoanTokens = repaymentBox.tokens(1)
		
		// Validate borrower's box
		val validBorrowerScript = borrowerScript == currentBorrower
		val validBorrowerCollateral = borrowerValue >= currentValue - MinimumTransactionFee

		// Validate repayment
		val validRepaymentScript = blake2b256(repaymentScript) == RepaymentContractScript
		val validRepaymentValue = repaymentValue >= MinimumBoxValue + MinimumTransactionFee
		val validRepaymentLoanTokens = repaymentLoanTokens._1 == PoolCurrencyId && repaymentLoanTokens._2 > totalOwed
		val validRecordOfLoan = repaymentBorrowTokens == currentBorrowTokens 

		// Check repayment conditions
		val repayment = (
			validBorrowerScript &&
			validBorrowerCollateral &&
			validRepaymentScript &&
			validRepaymentValue &&
			validRepaymentLoanTokens &&
			validRecordOfLoan &&
			validBaseChildNft &&
			validBaseChildIndex &&
			validParentNft &&
			validHeadChild 
		)
		val partialRepayment = if (OUTPUTS(0).tokens.size >= 1 && INPUTS(0).tokens.size < 3) {
			// Partial Repay
			// Extract values form successor
			val successor = OUTPUTS(0)
			val successorScript = successor.propositionBytes
			val successorValue = successor.value
			val successorBorrowTokens = successor.tokens(0)
			val successorBorrower = successor.R4[Coll[Byte]].get
			val successorIndexes = successor.R5[(Int, Int)].get
			val successorThresholdPenalty = successor.R6[(Long, Long)].get
			val successorDexNft = successor.R7[Coll[Byte]].get
			val successorUserPk = successor.R8[GroupElement].get
			val successorForcedLiquidation = successor.R9[Long].get
			
			// Extract values from dexBox
			val dexBox = CONTEXT.dataInputs(3)
			val dexReservesErg = dexBox.value
			val dexNft = dexBox.tokens(0)
			val dexReservesToken = dexBox.tokens(2)._2
			val inputAmount = successorValue - MaximumNetworkFee.toBigInt 
			val dexFee = dexBox.R4[Int].get		
			val collateralValue = (dexReservesToken.toBigInt * inputAmount.toBigInt * dexFee.toBigInt) /
			((dexReservesErg.toBigInt + (dexReservesErg.toBigInt * Slippage.toBigInt / 100.toBigInt)) * DexLpTaxDenomination.toBigInt +
			(inputAmount.toBigInt * dexFee.toBigInt)) 

			// Validate dexBox
			val validDexBox = dexNft._1 == currentDexNft
			
			val finalTotalOwed = successorBorrowTokens._2 * compoundedInterest / InterestRateDenom
			
			// Check sufficient collateral value to prevent double-spend attempts on partialRepayment
			val isSufficientCollateral = collateralValue >= finalTotalOwed.toBigInt * liquidationThreshold.toBigInt / LiquidationThresholdDenom.toBigInt
			
			// Calculate expected borrowTokens
			val repaymentMade = repaymentLoanTokens._2			
			val expectedBorrowTokens = currentBorrowTokens._2.toBigInt - (repaymentMade.toBigInt * InterestRateDenom.toBigInt / compoundedInterest.toBigInt)
			val validRepayLoanTokenId = repaymentLoanTokens._1 == PoolCurrencyId
			
			// Validate successor values
			val validSuccessorScript = successorScript == currentScript
			val retainMinValue = successorValue >= currentValue
			val retainBorrowTokenId = successorBorrowTokens._1 == currentBorrowTokens._1
			val validBorrowTokens = successorBorrowTokens._2.toBigInt >= expectedBorrowTokens
			val retainRegisters = (
				successorBorrower == currentBorrower &&
				successorIndexes == currentIndexes &&
				successorThresholdPenalty == currentThresholdPenalty &&
				successorDexNft == currentDexNft &&
				successorUserPk == currentUserPk &&
				successorForcedLiquidation == currentForcedLiquidation
			)

			// Validate repayment
			val validPartialRecordOfLoan = (
				repaymentBorrowTokens._2 == currentBorrowTokens._2 - successorBorrowTokens._2  &&
				repaymentBorrowTokens._1 == currentBorrowTokens._1
				)
			
			// Apply validation conditions 
			(
				validSuccessorScript &&
				retainMinValue &&
				retainBorrowTokenId &&
				validBorrowTokens &&
				retainRegisters &&
				validRepaymentScript &&
				validRepaymentValue &&
				validRepayLoanTokenId &&
				validPartialRecordOfLoan &&
				validBaseChildNft &&
				validBaseChildIndex &&
				validParentNft &&
				validHeadChild &&
				isSufficientCollateral &&
				validDexBox
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
			val inputAmount = currentValue - MaximumNetworkFee.toBigInt 
			val dexFee = dexBox.R4[Int].get		
			val collateralValue = (dexReservesToken.toBigInt * inputAmount.toBigInt * dexFee.toBigInt) /
			((dexReservesErg.toBigInt + (dexReservesErg.toBigInt * Slippage.toBigInt / 100.toBigInt)) * DexLpTaxDenomination.toBigInt +
			(inputAmount.toBigInt * dexFee.toBigInt)) 
		
			// Validate DEX box
			val validDexBox = dexBox.tokens(0)._1 == currentDexNft
			
			// Check if liquidation is allowed
			val liquidationAllowed = (
				collateralValue <= totalOwed.toBigInt * liquidationThreshold.toBigInt / LiquidationThresholdDenom.toBigInt || 
				HEIGHT > currentForcedLiquidation
			)
			
			val repaymentAmount = repaymentLoanTokens._2
			val validRepayLoanTokenId = repaymentLoanTokens._1 == PoolCurrencyId
			
			// Apply penalty on repayment and borrower share
			val borrowerShare = ((collateralValue - totalOwed.toBigInt) * (PenaltyDenom.toBigInt - liquidationPenalty.toBigInt)) / PenaltyDenom.toBigInt
			val applyPenalty = if (borrowerShare < 1.toBigInt) {
				repaymentAmount.toBigInt >= collateralValue
			} else {
				val validRepayment = repaymentAmount.toBigInt >= totalOwed.toBigInt + ((collateralValue - totalOwed.toBigInt) * liquidationPenalty.toBigInt / PenaltyDenom.toBigInt)
				val borrowBox = OUTPUTS(2)
				val validBorrowerShare = borrowBox.tokens(0)._2.toBigInt >= borrowerShare
				val validBorrowerShareId = borrowBox.tokens(0)._1 == PoolCurrencyId
				val validBorrowerAddress = borrowBox.propositionBytes == currentBorrower
				validRepayment && validBorrowerShare && validBorrowerAddress && validBorrowerShareId
			}   

			// Apply liquidation validations
			(
				liquidationAllowed &&
				validRepaymentScript &&
				validRepayLoanTokenId &&
				validRepaymentValue &&
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
	else {
		sigmaProp(false)
	}
}
```
