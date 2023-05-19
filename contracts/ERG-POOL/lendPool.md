```scala
{
	// Constants
	val CollateralContractScript = fromBase58("3vsAhAGr4TAqHH2upDn4EncoiZB7MJAGPfszuocrqDt5")
	val ChildBoxNft = fromBase58("3kMqXDumdsgdL1P1b3L3xDWe2tiLhW8uz8YoL8ano9Au")
	val ParamaterBoxNft = fromBase58("9M8VueZ5srjdUmi9T2sfkHCZfQPabFbr6EkEiBfP7LWn")
	val ParentBoxNft = fromBase58("2wGNGhTcrgRJRgBoQfdPiaT4Wvwx2FtDoqmtkJwWCXXj")
	val MaxLendTokens = 9000000001000000L // Set 1,000,000 higher than true maximum so that genesis lend token value is 1.
	val MaxBorrowTokens = 9000000000000000L
	val LiquidationThresholdDenomination = 10
	val MinimumBoxValue = 1000000 // Absolute minimum value allowed for pool box.
	val MinLoanValue = 50000000L
	val Slippage = 2
	val DexFee = 995
	val DexFeeDenom = 1000

	// Current pool values
	val currentScript = SELF.propositionBytes
	val currentPooledAssets = SELF.value
	val currentPoolNft = SELF.tokens(0)
	val currentReservedLendTokens = SELF.tokens(1)
	val currentBorrowTokens = SELF.tokens(2)
	val currentLendTokensCirculating = MaxLendTokens - currentReservedLendTokens._2
	val currentTotalBorrowed = MaxBorrowTokens - currentBorrowTokens._2

	// Successor pool values
	val successor = OUTPUTS(0)
	val successorScript = successor.propositionBytes
	val successorPooledAssets = successor.value
	val successorPoolNft = successor.tokens(0)
	val successorReservedLendTokens = successor.tokens(1)
	val successorBorrowTokens = successor.tokens(2)
	val successorLendTokensCirculating = MaxLendTokens - successorReservedLendTokens._2
	val successorTotalBorrowed = MaxBorrowTokens - successorBorrowTokens._2

	// Validation conditions under all spending paths
	val isValidSuccessorScript = successorScript == currentScript
	val isPoolNftPreserved = successorPoolNft == currentPoolNft 
	val isValidLendTokenId = successorReservedLendTokens._1 == currentReservedLendTokens._1 // Not required when the stronger isLendTokensUnchanged replaces this validation.
	val isValidBorrowTokenId = successorBorrowTokens._1 == currentBorrowTokens._1 // Not required when the stronger isBorrowTokensUnchanged replaces this validation.
	val isValidMinValue = successor.value >= MinimumBoxValue // Not required for a deposit since value must be increasing by isAssetsInPoolIncreasing

	// Calculate Lend Token valuation
	val currentLendTokenValue = (currentPooledAssets + currentTotalBorrowed) / currentLendTokensCirculating
	val successorLendTokenValue = (successorPooledAssets + successorTotalBorrowed) / successorLendTokensCirculating

	// Additional validation conditions specific to exchange operation
	val isLendTokenValueMaintained = successorLendTokenValue >= currentLendTokenValue // Ensures the current value of an LP token has not decreased
	val isBorrowTokensUnchanged = successorBorrowTokens == currentBorrowTokens

	// Validate exchange operation
	val isValidExchange = (
		isValidSuccessorScript &&
		isPoolNftPreserved &&
		isValidLendTokenId &&
		isValidMinValue &&
		isBorrowTokensUnchanged &&
		isLendTokenValueMaintained
	)

	// Allow for deposits
	val deltaTotalBorrowed = successorTotalBorrowed - currentTotalBorrowed
	val deltaAssetsInPool = successorPooledAssets - currentPooledAssets
	val deltaReservedLendTokens = successorReservedLendTokens._2 - currentReservedLendTokens._2

	val isLendTokensUnchanged = successorReservedLendTokens == currentReservedLendTokens
	val isAssetsInPoolIncreasing = deltaAssetsInPool > 0
	val isDepositDeltaBorrowedValid = deltaTotalBorrowed < 0 // Enforces a deposit to add lend tokens to the pool

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
		val loanAmount = collateralBorrowTokens._2
		val collateralParentIndex = collateralInterestIndexes(0)
		val collateralChildIndex = collateralInterestIndexes(1)
		
		// Load Dex pool values
		val dexBox = CONTEXT.dataInputs(0)
		val dexReservesErg = dexBox.value
		val dexNft = dexBox.tokens(0)
		val dexReservesToken = dexBox.tokens(2)._2

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
		val indexOfDexBoxNFT = collateralDexNfts.indexOf(dexNft._1, 0)
		val indexOfAssetId = collateralAssetIds.indexOf(collateralSupplied._1, 0)
		val liquidationThreshold = liquidationThresholds(indexOfAssetId)
		val liquidationPenalty = liquidationPenalties(indexOfAssetId)

		// Check sufficient collateral
		val inputAmount = collateralSupplied._2
		val collateralMarketValue = (dexReservesErg * inputAmount * DexFee) /
			((dexReservesToken + (dexReservesToken * Slippage / 100)) * DexFeeDenom +
			(inputAmount * DexFee / DexFeeDenom)) // Formula from spectrum dex
		val isCorrectCollateralAmount = collateralMarketValue >= loanAmount * liquidationThreshold / LiquidationThresholdDenomination
			
		// Check correctly mapped DexNft
		val isAssetMapsToDexNft = (indexOfDexBoxNFT == indexOfAssetId) && indexOfAssetId >= 0 // For when both index searches return not found (-1)
		val isValidCollateral = isAssetMapsToDexNft && isCorrectCollateralAmount
		val isValidCollateralBorrowTokenId = collateralBorrowTokens._1 == currentBorrowTokens._1

		// Check if interest indexes, penalty array, asset ID, and DEX NFT are valid in collateralBox
		val isValidChildIndex = collateralChildIndex == childInterestHistory.size - 1
		val isValidParentIndex = collateralParentIndex == parentInterestHistory.size
		val isValidThresholdPenaltyArray = collateralThresholdPenalty == (liquidationThreshold, liquidationPenalty)
		val isValidDexNft = collateralDexNft == dexNft._1

		// Validate Erg and token values in LendPool and collateralBox
		val isAssetAmountValid = deltaAssetsInPool * -1 == loanAmount
		val isTotalBorrowedValid = deltaTotalBorrowed == loanAmount
		val isSufficientLoanValue = loanAmount >= MinLoanValue // Cover dexFee loss in case of immediate liquidation
		
		// Validate other loaded boxes
		val isValidCollateralContract = blake2b256(collateralScript) == CollateralContractScript
		val isValidCollateralValue = collateralValue >= MinimumBoxValue // Ensure collateral value sufficient for safety
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
