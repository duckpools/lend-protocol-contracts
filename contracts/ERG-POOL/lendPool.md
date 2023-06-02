```scala
{
	// Constants
	val CollateralContractScript = fromBase58("JCDEqzXJYbiRyut9amJSvt1uebWRwqtUSMmqYyUn9x3A")
	val ChildBoxNft = fromBase58("3kMqXDumdsgdL1P1b3L3xDWe2tiLhW8uz8YoL8ano9Au")
	val ParamaterBoxNft = fromBase58("9M8VueZ5srjdUmi9T2sfkHCZfQPabFbr6EkEiBfP7LWn")
	val ParentBoxNft = fromBase58("2wGNGhTcrgRJRgBoQfdPiaT4Wvwx2FtDoqmtkJwWCXXj")
	val MaxLendTokens = 9000000001000000L // Set 1,000,000 higher than true maximum so that genesis lend token value is 1.
	val MaxBorrowTokens = 9000000000000000L
	val LiquidationThresholdDenomination = 10
	val MinimumBoxValue = 1000000 // Absolute minimum value allowed for pool box.
	val MinimumTxFee = 1000000L
	val MinLoanValue = 50000000L
	val Slippage = 2
	val DexFeeDenom = 1000
	val sFeeStepOne = 20000000000L
	val sFeeStepTwo = 200000000000L
	val sFeeDivisorOne = 160
	val sFeeDivisorTwo = 200
	val sFeeDivisorThree = 250
	val LendTokenMultipler = 1000000000000000L.toBigInt
	val MaximumNetworkFee = 4000000
	val forcedLiquidationBuffer = 500000

	// Current pool values
	val currentScript = SELF.propositionBytes
	val currentPooledAssets = SELF.value
	val currentPoolNft = SELF.tokens(0)
	val currentLendTokens = SELF.tokens(1)
	val currentBorrowTokens = SELF.tokens(2)
	val currentLendTokensCirculating = MaxLendTokens - currentLendTokens._2
	val currentTotalBorrowed = MaxBorrowTokens - currentBorrowTokens._2

	// Successor pool values
	val successor = OUTPUTS(0)
	val successorScript = successor.propositionBytes
	val successorPooledAssets = successor.value
	val successorPoolNft = successor.tokens(0)
	val successorLendTokens = successor.tokens(1)
	val successorBorrowTokens = successor.tokens(2)
	val successorLendTokensCirculating = MaxLendTokens - successorLendTokens._2
	val successorTotalBorrowed = MaxBorrowTokens - successorBorrowTokens._2

	// Validation conditions under all spending paths
	val isValidSuccessorScript = successorScript == currentScript
	val isPoolNftPreserved = successorPoolNft == currentPoolNft 
	val isValidLendTokenId = successorLendTokens._1 == currentLendTokens._1 // Not required when the stronger isLendTokensUnchanged replaces this validation.
	val isValidBorrowTokenId = successorBorrowTokens._1 == currentBorrowTokens._1 // Not required when the stronger isBorrowTokensUnchanged replaces this validation.
	val isValidMinValue = successor.value >= MinimumBoxValue // Not required for a deposit since value must be increasing by isAssetsInPoolIncreasing

	// Calculate Lend Token valuation
	val currentLendTokenValue = LendTokenMultipler * (currentPooledAssets.toBigInt + currentTotalBorrowed.toBigInt) / currentLendTokensCirculating.toBigInt
	val successorLendTokenValue = LendTokenMultipler * (successorPooledAssets.toBigInt + successorTotalBorrowed.toBigInt) / successorLendTokensCirculating.toBigInt

	// Additional validation conditions specific to exchange operation
	val isLendTokenValueMaintained = successorLendTokenValue > currentLendTokenValue // Ensures the current value of an LP token has not decreased
	val isBorrowTokensUnchanged = successorBorrowTokens == currentBorrowTokens
	
	// Useful value to store
	val deltaAssetsInPool = successorPooledAssets - currentPooledAssets
	val absDeltaAssetsInPool = if (deltaAssetsInPool > 0) deltaAssetsInPool else deltaAssetsInPool * -1
	
	// Validate service fee
	val isValidServiceFee = if (CONTEXT.dataInputs.size > 0) {
		// Extract values from service fee box
		val serviceFeeBox = OUTPUTS(1)
		val serviceFeeScript = serviceFeeBox.propositionBytes
		val serviceFeeValue = serviceFeeBox.value
		
		// Extract values form serviceParamBox
		val serviceParamBox = CONTEXT.dataInputs(0)
		val serviceParamNft = serviceParamBox.tokens(0)
		val paramServiceFeeScript = serviceParamBox.R8[Coll[Byte]].get
		
		// Calculate total service fee owed
		val totalServiceFee = if (absDeltaAssetsInPool <= sFeeStepOne) {
			absDeltaAssetsInPool / sFeeDivisorOne
		} else {
			if (absDeltaAssetsInPool <= sFeeStepTwo) {
				(absDeltaAssetsInPool - sFeeStepOne) / sFeeDivisorTwo +
				sFeeStepOne / sFeeDivisorOne
			} else {
				(absDeltaAssetsInPool - sFeeStepTwo - sFeeStepOne) / sFeeDivisorThree +
				sFeeStepTwo / sFeeDivisorTwo +
				sFeeStepOne / sFeeDivisorOne
			}
		}
			
		// Validate service fee box
		val validServiceScript = serviceFeeScript == paramServiceFeeScript
		val validServiceFeeValue = serviceFeeValue >= max(totalServiceFee, MinimumBoxValue)
	
		// Validate serviceParamBox
		val isValidServiceParamBox = serviceParamNft._1 == ParamaterBoxNft
		
		// Apply validation conditions
		(
			validServiceScript &&
			validServiceFeeValue &&
			isValidServiceParamBox
		)
	} else {
		false
	}

	// Validate exchange operation
	val isValidExchange = (
		isValidSuccessorScript &&
		isPoolNftPreserved &&
		isValidLendTokenId &&
		isValidMinValue &&
		isBorrowTokensUnchanged &&
		isLendTokenValueMaintained &&
		isValidServiceFee
	)

	// Allow for deposits
	val deltaTotalBorrowed = successorTotalBorrowed - currentTotalBorrowed
	val deltaReservedLendTokens = successorLendTokens._2 - currentLendTokens._2

	val isLendTokensUnchanged = successorLendTokens == currentLendTokens
	val isAssetsInPoolIncreasing = deltaAssetsInPool > 0
	val isDepositDeltaBorrowedValid = deltaTotalBorrowed < 0 // Enforces a deposit to add borrow tokens to the pool

	// Validate deposit operation
	val isValidDeposit = (
		isValidSuccessorScript &&
		isPoolNftPreserved &&
		isLendTokensUnchanged &&
		isValidBorrowTokenId &&
		isAssetsInPoolIncreasing &&
		isDepositDeltaBorrowedValid
	)

	// Check for loans
	if (CONTEXT.dataInputs.size > 0) {
		// Load Collateral Box Values
		val collateralBox = OUTPUTS(1)
		val collateralScript = collateralBox.propositionBytes
		val collateralValue = collateralBox.value
		val collateralSupplied = collateralBox.tokens(0)
		val collateralBorrowTokens = collateralBox.tokens(1)
		val isCollateralOwnerDefined = collateralBox.R4[Coll[Byte]].isDefined
		val collateralInterestIndexes = collateralBox.R5[(Int, Int)].get
		val collateralThresholdPenalty = collateralBox.R6[(Long, Long)].get
		val collateralDexNft = collateralBox.R7[Coll[Byte]].get
		val isUserPkDefined = collateralBox.R8[GroupElement].isDefined
		val forcedLiquidation = collateralBox.R9[Long].get
		val loanAmount = collateralBorrowTokens._2
		val collateralParentIndex = collateralInterestIndexes(0)
		val collateralChildIndex = collateralInterestIndexes(1)
		
		// Load Dex pool values
		val dexBox = CONTEXT.dataInputs(0)
		val dexReservesErg = dexBox.value
		val dexNft = dexBox.tokens(0)
		val dexReservesToken = dexBox.tokens(2)
		val dexFee = dexBox.R4[Int].get

		// Load parent box values
		val parentInterestBox = CONTEXT.dataInputs(1)
		val parentBoxNft = parentInterestBox.tokens(0)
		val parentInterestHistory = parentInterestBox.R4[Coll[Long]].get
		
		// Load child interest box values
		val childInterestBox = CONTEXT.dataInputs(2)
		val childBoxNft = childInterestBox.tokens(0)
		val childInterestHistory = childInterestBox.R4[Coll[Long]].get
		val childIndex = childInterestBox.R6[Int].get

		// Load parameter box values
		val paramaterBox = CONTEXT.dataInputs(3)
		val paramaterNft = paramaterBox.tokens(0)
		val liquidationThresholds = paramaterBox.R4[Coll[Long]].get
		val collateralAssetIds = paramaterBox.R5[Coll[Coll[Byte]]].get
		val collateralDexNfts = paramaterBox.R6[Coll[Coll[Byte]]].get
		val liquidationPenalties = paramaterBox.R7[Coll[Long]].get
		
		// Get collateral settings from paramaterBox
		val indexOfParams = collateralDexNfts.indexOf(dexNft._1, 0)
		val expectedCollateralId = collateralAssetIds(indexOfParams)
		val liquidationThreshold = liquidationThresholds(indexOfParams)
		val liquidationPenalty = liquidationPenalties(indexOfParams)

		// Check sufficient collateral
		val inputAmount = collateralSupplied._2
		val collateralMarketValue = (dexReservesErg * inputAmount * dexFee) /
			((dexReservesToken._2 + (dexReservesToken._2 * Slippage / 100)) * DexFeeDenom +
			(inputAmount * dexFee)) - MaximumNetworkFee // Formula from spectrum dex
		val isCorrectCollateralAmount = collateralMarketValue >= loanAmount * liquidationThreshold / LiquidationThresholdDenomination
			
		// Check correct collateral box tokens
		val isValidCollateralId = (dexReservesToken._1 == collateralSupplied._1) && (collateralSupplied._1 == expectedCollateralId)
		val isValidCollateral = isValidCollateralId && isCorrectCollateralAmount
		val isValidCollateralBorrowTokenId = collateralBorrowTokens._1 == currentBorrowTokens._1

		// Check if interest indexes, penalty array, asset ID, and DEX NFT are valid in collateralBox
		val isValidChildIndex = collateralChildIndex == childInterestHistory.size - 1
		val isValidParentIndex = collateralParentIndex == parentInterestHistory.size
		val isValidThresholdPenaltyArray = collateralThresholdPenalty == (liquidationThreshold, liquidationPenalty)
		val isValidDexNft = collateralDexNft == dexNft._1
		
		// Check forced liquidation height
		val isValidForcedLiquidation = forcedLiquidation > HEIGHT && forcedLiquidation < HEIGHT + forcedLiquidationBuffer

		// Validate Erg and token values in LendPool and collateralBox
		val isAssetAmountValid = deltaAssetsInPool * -1 == loanAmount
		val isTotalBorrowedValid = deltaTotalBorrowed == loanAmount
		val isSufficientLoanValue = loanAmount >= MinLoanValue // Cover dexFee loss in case of immediate liquidation
		
		// Validate other loaded boxes
		val isValidCollateralContract = blake2b256(collateralScript) == CollateralContractScript
		val isValidCollateralValue = collateralValue >= MinimumBoxValue + MinimumTxFee// Ensure collateral value sufficient for safety
		val isValidChildBox = childBoxNft._1 == ChildBoxNft && childIndex == parentInterestHistory.size
		val isValidParentBox = parentBoxNft._1 == ParentBoxNft
		val isValidParamaterBox = paramaterNft._1 == ParamaterBoxNft

		// Validate borrow operation
		val isValidBorrow = (
			isValidSuccessorScript &&
			isAssetAmountValid &&
			isValidMinValue &&
			isPoolNftPreserved &&
			isLendTokensUnchanged &&
			isValidBorrowTokenId &&
			isTotalBorrowedValid &&
			isValidCollateralContract &&
			isValidCollateralValue &&
			isValidCollateral &&
			isValidCollateralBorrowTokenId &&
			isSufficientLoanValue &&
			isCollateralOwnerDefined &&
			isValidChildIndex &&
			isValidParentIndex &&
			isValidThresholdPenaltyArray &&
			isValidDexNft &&
			isUserPkDefined &&
			isValidForcedLiquidation &&
			isValidChildBox &&
			isValidParentBox &&
			isValidParamaterBox
		)
		sigmaProp(isValidBorrow)
	} else {
		sigmaProp(isValidDeposit || isValidExchange)
	}
}
```
