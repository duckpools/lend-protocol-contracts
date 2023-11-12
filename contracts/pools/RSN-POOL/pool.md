```scala
{
	// Constants
	val CollateralContractScript = fromBase58("B5V5Bu5hKXZiYrMAs628i1jjHmgva7LfxRx3zzHgxqAG")
	val ChildBoxNft = fromBase58("DWvuhYpDdvP322QqyUdNKAgT79VjbzN1yoM3QCWEbRuF")
	val ParamaterBoxNft = fromBase58("BNupr5uTP4eJr5h9agvwD3fUBq88SKt2icsCjapcWDf9")
	val ParentBoxNft = fromBase58("TsBqrGRa3m743ZKZyNws8oqyjdXTm1yWBP79b9ojvd9")
	val MaxLendTokens = 9000000000000010L // Set 1,000,000 higher than true maximum so that genesis lend token value is 1.
	val MaxBorrowTokens = 9000000000000000L
	val LiquidationThresholdDenomination = 1000
	val MinimumBoxValue = 1000000 // Absolute minimum value allowed for pool box.
	val MinimumTxFee = 1000000L
	val Slippage = 2
	val DexFeeDenom = 1000
	val sFeeStepOne = 400000L
	val sFeeStepTwo = 4000000L
	val sFeeDivisorOne = 160
	val sFeeDivisorTwo = 200
	val sFeeDivisorThree = 250
	val LendTokenMultipler = 1000000000000000L.toBigInt
	val MaximumNetworkFee = 5000000
	val MinLoanValue = 20000L
	val forcedLiquidationBuffer = 500000
	val defaultBuffer = 100000000L

	// Current pool values
	val currentScript = SELF.propositionBytes
	val currentPoolValue = SELF.value
	val currentPoolNft = SELF.tokens(0)
	val currentLendTokens = SELF.tokens(1)
	val currentBorrowTokens = SELF.tokens(2)
	val currentPooledTokens = SELF.tokens(3)
	val currentLendTokensCirculating = MaxLendTokens - currentLendTokens._2
	val currentTotalBorrowed = MaxBorrowTokens - currentBorrowTokens._2
	val currentPooledAssets = currentPooledTokens._2

	// Successor pool values
	val successor = OUTPUTS(0)
	val successorScript = successor.propositionBytes
	val successorPoolValue = successor.value
	val successorPoolNft = successor.tokens(0)
	val successorLendTokens = successor.tokens(1)
	val successorBorrowTokens = successor.tokens(2)
	val successorPooledTokens = successor.tokens(3)
	val successorLendTokensCirculating = MaxLendTokens - successorLendTokens._2
	val successorTotalBorrowed = MaxBorrowTokens - successorBorrowTokens._2
	val successorPooledAssets = successorPooledTokens._2

	// Validation conditions under all spending paths
	val isValidSuccessorScript = successorScript == currentScript
	val isPoolNftPreserved = successorPoolNft == currentPoolNft 
	val isValidLendTokenId = successorLendTokens._1 == currentLendTokens._1 // Not required when the stronger isLendTokensUnchanged replaces this validation.
	val isValidBorrowTokenId = successorBorrowTokens._1 == currentBorrowTokens._1 // Not required when the stronger isBorrowTokensUnchanged replaces this validation.
	val isValidMinValue = successorPoolValue >= MinimumBoxValue 
	val isPooledAssetPreserved = successorPooledTokens._1 == currentPooledTokens._1

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
		val serviceFeeTokens = serviceFeeBox.tokens(0)
		val serviceFeeAssets = serviceFeeTokens._2
		
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
		val validServiceFeeValue = serviceFeeValue >= MinimumBoxValue
		val validServiceTokens = serviceFeeAssets >= max(totalServiceFee, 1L) && serviceFeeTokens._1 == currentPooledTokens._1
	
		// Validate serviceParamBox
		val isValidServiceParamBox = serviceParamNft._1 == ParamaterBoxNft
		
		// Apply validation conditions
		(
			validServiceScript &&
			validServiceFeeValue &&
			validServiceTokens &&
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
		isPooledAssetPreserved &&
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
		isValidMinValue &&
		isPoolNftPreserved &&
		isLendTokensUnchanged &&
		isValidBorrowTokenId &&
		isAssetsInPoolIncreasing &&
		isPooledAssetPreserved &&
		isDepositDeltaBorrowedValid
	)

	// Check for loans
	if (CONTEXT.dataInputs.size > 1) {
		// Load Collateral Box Values
		val collateralBox = OUTPUTS(1)
		val collateralScript = collateralBox.propositionBytes
		val collateralValue = collateralBox.value
		val collateralBorrowTokens = collateralBox.tokens(0)
		val isCollateralOwnerDefined = collateralBox.R4[Coll[Byte]].isDefined
		val collateralInterestIndexes = collateralBox.R5[(Int, Int)].get
		val collateralThresholdPenalty = collateralBox.R6[(Long, Long)].get
		val collateralDexNft = collateralBox.R7[Coll[Byte]].get
		val isUserPkDefined = collateralBox.R8[GroupElement].isDefined
		val forcedLiquidation = collateralBox.R9[(Long, Long)].get._1
		val bufferLiquidation = collateralBox.R9[(Long, Long)].get._2
		val loanAmount = collateralBorrowTokens._2
		val collateralParentIndex = collateralInterestIndexes(0)
		val collateralChildIndex = collateralInterestIndexes(1)
		
		// Load Dex pool values
		val poolNativeCurrencyId = currentPooledTokens._1
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
		val inputAmount = collateralValue - MaximumNetworkFee.toBigInt 
		val collateralMarketValue = (dexReservesToken._2.toBigInt * inputAmount.toBigInt * dexFee.toBigInt) /
			((dexReservesErg.toBigInt + (dexReservesErg.toBigInt * Slippage.toBigInt / 100.toBigInt)) * DexFeeDenom.toBigInt +
			(inputAmount.toBigInt * dexFee.toBigInt)) 
		
		val isCorrectCollateralAmount = collateralMarketValue >= loanAmount.toBigInt * liquidationThreshold.toBigInt / LiquidationThresholdDenomination.toBigInt
			
		// Check correct collateral box tokens
		val isValidCollateral = isCorrectCollateralAmount
		val isValidCollateralBorrowTokenId = collateralBorrowTokens._1 == currentBorrowTokens._1

		// Check if interest indexes, penalty array, asset ID, and DEX NFT are valid in collateralBox
		val isValidChildIndex = collateralChildIndex == childInterestHistory.size - 1
		val isValidParentIndex = collateralParentIndex == parentInterestHistory.size
		val isValidThresholdPenaltyArray = collateralThresholdPenalty == (liquidationThreshold, liquidationPenalty)
		val isValidDexNft = collateralDexNft == dexNft._1
		
		// Check forced liquidation height
		val isValidForcedLiquidation = forcedLiquidation > HEIGHT && forcedLiquidation < HEIGHT + forcedLiquidationBuffer
		val isValidLiquidationBuffer = bufferLiquidation == defaultBuffer

		// Validate Erg and token values in LendPool and collateralBox
		val isAssetAmountValid = deltaAssetsInPool * -1 == loanAmount
		val isTotalBorrowedValid = deltaTotalBorrowed == loanAmount
		
		// Validate other loaded boxes
		val isValidCollateralContract = blake2b256(collateralScript) == CollateralContractScript
		val isValidCollateralValue = collateralValue >= MinLoanValue + MinimumBoxValue + MinimumTxFee // Ensure collateral value sufficient for safety
		val isValidChildBox = childBoxNft._1 == ChildBoxNft && childIndex == parentInterestHistory.size
		val isValidParentBox = parentBoxNft._1 == ParentBoxNft
		val isValidParamaterBox = paramaterNft._1 == ParamaterBoxNft
		
		// Ensure asset reduction occurs
		val isAssetsInPoolDecreasing = deltaAssetsInPool < 0
		
		val isValidBaseToken = dexReservesToken._1 == poolNativeCurrencyId

		// Validate borrow operation
		val isValidBorrow = (
			isValidSuccessorScript &&
			isAssetAmountValid &&
			isAssetsInPoolDecreasing &&
			isValidMinValue &&
			isPoolNftPreserved &&
			isPooledAssetPreserved &&
			isLendTokensUnchanged &&
			isValidBorrowTokenId &&
			isTotalBorrowedValid &&
			isValidCollateralContract &&
			isValidCollateralValue &&
			isValidCollateral &&
			isValidCollateralBorrowTokenId &&
			isCollateralOwnerDefined &&
			isValidChildIndex &&
			isValidParentIndex &&
			isValidThresholdPenaltyArray &&
			isValidDexNft &&
			isUserPkDefined &&
			isValidForcedLiquidation &&
			isValidLiquidationBuffer &&
			isValidChildBox &&
			isValidBaseToken &&
			isValidParentBox &&
			isValidParamaterBox
		)
		sigmaProp(isValidBorrow)
	} else {
		sigmaProp(isValidDeposit || isValidExchange)
	}
}
```
