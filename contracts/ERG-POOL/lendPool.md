```scala
{
   // Constants
    val CollateralContractScript = fromBase58("BiJiVLmKKPg5xbsxyu8jyzsYeWxuD4oaUkQcqcAak9r2")
    val InterestBoxNft = fromBase58("7vkbEYNKCdtBAQy62YdoegYzNTAykVAcfvToNwz65FhZ")
    val ParamaterBoxNft = fromBase58("9M8VueZ5srjdUmi9T2sfkHCZfQPabFbr6EkEiBfP7LWn")
    val MaxLendTokens = 9000000001000000L // Set 1,000,000 higher than true maximum so that genesis lend token value is 1.
    val LiquidationThresholdDenomination = 10
    val CollectionElementNotFound = -1
    val MinimumBoxValue = 1000000 // Absolute minimum value allowed for pool box.
    val MinLoanValue = 50000000L

    // Current pool values
    val currentPoolNft = SELF.tokens(0)
    val currentReservedLendTokens = SELF.tokens(1)
    val currentTotalBorrowed = SELF.R4[Long].get

    // Successor pool values
    val successor = OUTPUTS(0)
    val successorPoolNft = successor.tokens(0)
    val successorReservedLendTokens = successor.tokens(1)
    val successorTotalBorrowed = successor.R4[Long].get

    // Validation conditions for successor pool box under all spending paths
    val isValidSuccessorScript = successor.propositionBytes == SELF.propositionBytes
    val isPoolNftPreserved = successorPoolNft == currentPoolNft // Not required when the stronger isPoolTokensUnchanged replaces this validation.
    val isValidLendTokenId = successorReservedLendTokens._1 == currentReservedLendTokens._1 // Not required when the stronger isPoolTokensUnchanged replaces this validation.
    val noAdditionalTokens = successor.tokens.size == 2 // Not required when the stronger isPoolTokensUnchanged replaces this validation.
    val isValidMinValue = successor.value >= MinimumBoxValue // Not required for a deposit since value must be increasing by isAssetsInPoolIncreasing

    // Calculate lend token supply and held assets for initial and successor pool boxes
    val currentLendTokensCirculating = MaxLendTokens - currentReservedLendTokens._2
    val successorLendTokensCirculating = MaxLendTokens - successorReservedLendTokens._2
    val currentAssetsInPool = SELF.value
    val successorAssetsInPool = successor.value

    // Calculate delta and LP values for current and successor pool boxes
    val deltaTotalBorrowed = successorTotalBorrowed - currentTotalBorrowed
    val isDeltaBorrowedValid = deltaTotalBorrowed == 0
    val currentLendTokenValue = (currentAssetsInPool + currentTotalBorrowed) / currentLendTokensCirculating
    val successorLendTokenValue = (successorAssetsInPool + successorTotalBorrowed) / successorLendTokensCirculating
    val isLendTokenValueMaintained = successorLendTokenValue >= currentLendTokenValue // Ensures the current value of an LP token has not decreased

    // Validate exchange operation
    val isValidExchange = (
        isValidSuccessorScript &&
        isPoolNftPreserved &&
        isValidLendTokenId &&
        noAdditionalTokens &&
        isValidMinValue &&
        isDeltaBorrowedValid &&
        isLendTokenValueMaintained
    )

    // Allow for deposits
    val deltaAssetsInPool = successorAssetsInPool - currentAssetsInPool
    val deltaReservedLendTokens = successorReservedLendTokens._2 - currentReservedLendTokens._2

    val isAssetsInPoolIncreasing = deltaAssetsInPool > 0
    val isPoolTokensUnchanged = successor.tokens == SELF.tokens
    val isDepositDeltaBorrowedValid = deltaAssetsInPool >= deltaTotalBorrowed * -1 || (currentTotalBorrowed - deltaAssetsInPool < 0 && successorTotalBorrowed == 0) // Ensures LP value cannot decrease
    val isMinBorrowValue = successorTotalBorrowed >= 0 // Likely redudant, but this gurantees total borrowed is non-negative

    // Validate deposit operation
    val isValidDeposit = (
        isValidSuccessorScript &&
        isPoolTokensUnchanged &&
        isAssetsInPoolIncreasing &&
        isDepositDeltaBorrowedValid &&
        isMinBorrowValue
    )

    // Check for loans
    if (CONTEXT.dataInputs.size > 0) {
        // Loan-related boxes
        val collateralBox = OUTPUTS(1)
        val dexBox = CONTEXT.dataInputs(0)
        val interestBox = CONTEXT.dataInputs(1)
        val paramaterBox = CONTEXT.dataInputs(2)

        // Get parameter values from paramaterBox
        val liquidationThresholds = paramaterBox.R4[Coll[Long]].get
        val collateralAssetIds = paramaterBox.R5[Coll[Coll[Byte]]].get
        val collateralDexNfts = paramaterBox.R6[Coll[Coll[Byte]]].get
        val liquidationPenalties = paramaterBox.R7[Coll[Long]].get

        val loanAmount = collateralBox.R4[Long].get

        // Dex pool values
        val ergInDexPool = dexBox.value
        val tokensInDexPool = dexBox.tokens(2)._2
        val collateralNanoErgsPerToken = ergInDexPool / tokensInDexPool

        // Interest history
        val interestHistory = interestBox.R4[Coll[Long]].get

        // Validation conditions for loan-related boxes
        val isValidCollateralContract = blake2b256(collateralBox.propositionBytes) == CollateralContractScript
        val isValidInterestBox = interestBox.tokens(0)._1 == InterestBoxNft
        val isValidParamaterBox = paramaterBox.tokens(0)._1 == ParamaterBoxNft

        // Indices of collateral settings from paramaterBox 
        val indexOfDexBoxNFT = collateralDexNfts.indexOf(dexBox.tokens(0)._1, 0)
        val indexOfAssetId = if (collateralBox.tokens.size != 0) {
            collateralAssetIds.indexOf(collateralBox.tokens(0)._1, 0)
        } else {
            CollectionElementNotFound
        }
        val liquidationThreshold = liquidationThresholds(indexOfAssetId)
        val liquidationPenalty = liquidationPenalties(indexOfAssetId)

        // Check if asset correctly maps to DEX NFT and collateral amount is correct
        val isAssetMapsToDexNft = (indexOfDexBoxNFT == indexOfAssetId) && indexOfAssetId >= 0
        val isCorrectCollateralAmount = if (collateralBox.tokens.size != 0) {
            collateralBox.tokens(0)._2 >= (loanAmount * liquidationThreshold) / (collateralNanoErgsPerToken * LiquidationThresholdDenomination)
        } else {
            false
        }

        val isValidCollateral = isAssetMapsToDexNft && isCorrectCollateralAmount

        // Check if interest index, penalty array, asset ID, and DEX NFT are valid in collateralBox
        val isValidInterestIndex = collateralBox.R6[Int].get == interestHistory.size - 1
        val isValidThresholdPenaltyArray = collateralBox.R7[Coll[Long]].get == Coll(liquidationThreshold, liquidationPenalty)
        val isValidDexNft = collateralBox.R8[Coll[Byte]].get == dexBox.tokens(0)._1

        // Validate Erg and token values in LendPool and collateralBox
        val isAssetAmountValid = deltaAssetsInPool * -1 == loanAmount
        val isTotalBorrowedValid = deltaTotalBorrowed == loanAmount
        val isSufficientLoanValue = loanAmount >= MinLoanValue


        // Check for compilation error prevention in collateralBox
        val isValidCollateralBoxR5 = collateralBox.R5[Coll[Byte]].isDefined

        // Validate borrow operation
        val IsValidBorrow = (
            isValidSuccessorScript &&
            isAssetAmountValid &&
            isPoolTokensUnchanged &&
            isTotalBorrowedValid &&
            isValidCollateralContract &&
            isValidCollateralBoxR5 &&
            isValidInterestBox &&
            isValidCollateral &&
            isValidInterestIndex &&
            isValidThresholdPenaltyArray &&
            isValidDexNft &&
            isSufficientLoanValue &&
            isValidMinValue
        )
        sigmaProp(IsValidBorrow)
    } else {
        sigmaProp(isValidDeposit || isValidExchange)
    }
}

