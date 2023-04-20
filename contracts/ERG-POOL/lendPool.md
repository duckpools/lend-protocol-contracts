```scala
{
   // Constants
    val CollateralContractScript = fromBase58("21oFkHsLgX7ZwQDKrQFwe69o3LyJvqSC2MFTfuuNtMEX")
    val InterestBoxNft = fromBase58("7vkbEYNKCdtBAQy62YdoegYzNTAykVAcfvToNwz65FhZ")
    val ParamaterBoxNft = fromBase58("9M8VueZ5srjdUmi9T2sfkHCZfQPabFbr6EkEiBfP7LWn")
    val MaxLendTokens = 9000000001000000L // Set 1,000,000 higher than true maximum so that genesis lend token value is 1.
    val MaxBorrowTokens = 9000000000000000L
    val LiquidationThresholdDenomination = 10
    val CollectionElementNotFound = -1
    val MinimumBoxValue = 1000000 // Absolute minimum value allowed for pool box.
    val MinLoanValue = 50000000L

    // Current pool values
    val currentPoolNft = SELF.tokens(0)
    val currentReservedLendTokens = SELF.tokens(1)
    val currentBorrowTokens = SELF.tokens(2)
    val currentTotalBorrowed = MaxBorrowTokens - currentBorrowTokens._2

    // Successor pool values
    val successor = OUTPUTS(0)
    val successorPoolNft = successor.tokens(0)
    val successorReservedLendTokens = successor.tokens(1)
    val successorBorrowTokens = successor.tokens(2)
    val successorTotalBorrowed = MaxBorrowTokens - successorBorrowTokens._2

    // Validation conditions for successor pool box under all spending paths
    val isValidSuccessorScript = successor.propositionBytes == SELF.propositionBytes
    val isPoolNftPreserved = successorPoolNft == currentPoolNft // Not required when the stronger isPoolTokensUnchanged replaces this validation.
    val isValidLendTokenId = successorReservedLendTokens._1 == currentReservedLendTokens._1 // Not required when the stronger isPoolTokensUnchanged replaces this validation.
    val isValidBorrowTokenId = successorBorrowTokens._1 == currentBorrowTokens._1
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
        isValidBorrowTokenId &&
        isValidMinValue &&
        isDeltaBorrowedValid &&
        isLendTokenValueMaintained
    )

    // Allow for deposits
    val deltaAssetsInPool = successorAssetsInPool - currentAssetsInPool
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

        val collateralBorrowTokens = collateralBox.tokens(1)
        val loanAmount = collateralBorrowTokens._2

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
        val isValidCollateralBorrowTokenId = collateralBorrowTokens._1 == currentBorrowTokens._1

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
            isPoolNftPreserved &&
            isLendTokensUnchanged &&
            isValidBorrowTokenId &&
            isTotalBorrowedValid &&
            isValidCollateralContract &&
            isValidCollateralBoxR5 &&
            isValidInterestBox &&
            isValidCollateral &&
            isValidCollateralBorrowTokenId &&
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
```
