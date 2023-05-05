```scala
{
    // Constants
    val RepaymentContractScript = fromBase58("HFhgYSWt87paJWygPTQfuGeysS88Ba8TuqziAQvSiRWv")
    val InterestBoxNft = fromBase58("BXV2vpmMwV5PkJoK2EMA5nyM5rGPd7oWRK46fhLbD91")
    val InterestContractDenomination = 1000000L
    val MaximumNetworkFee = 4000000
    val DexLpTax = 993 // 99.3% retained
    val DexLpTaxDenomination = 1000 
    val LiquidationThresholdDenomination = 10
    val penaltyDenomination = 10
    val MinimumTransactionFee = 1000000L
    val ForcedLiquidationHeight = 2000000
    val minimumBoxValue = 1000000
	
	// Extract variables from SELF
    val currentScript = SELF.propositionBytes
    val currentCollateral = SELF.tokens(0)
    val currentBorrowTokens = SELF.tokens(1)
    val loanAmount = currentBorrowTokens._2
    val currentBorrower = SELF.R4[Coll[Byte]].get
    val currentIndex = SELF.R5[Int].get
    val currentThresholdPenalty = SELF.R6[Coll[Long]].get
    val liquidationThreshold = currentThresholdPenalty(0)
    val liquidationPenalty = currentThresholdPenalty(1)
    val currentDexNft = SELF.R7[Coll[Byte]].get
    val currentUserPk = SELF.R8[GroupElement].get

    // Extract values from interestBox
    val interestBox = CONTEXT.dataInputs(0)
    val historicalRates = interestBox.R4[Coll[Long]].get
    val historySize = historicalRates.size
    val historyToScan = historicalRates.slice(currentIndex, historySize)
    val compoundedInterest = historyToScan.fold(InterestContractDenomination, {(z:Long, base:Long) => z + base - InterestContractDenomination})
    val totalOwed = loanAmount * compoundedInterest / InterestContractDenomination

	// Validate Interest Box
    val validInterestNFT = interestBox.tokens(0)._1 == InterestBoxNft

	// Branch into collateral-top ups and repayment/ liquidation
    if(INPUTS(0) == SELF) {
        // Branch for adjusting collateral levels
        // Get values from successor collateral box
        val successor = OUTPUTS(0)
        val successorScript = successor.propositionBytes
        val successorCollateral = successor.tokens(0)
        val successorBorrowTokens = successor.tokens(1)
        val successorBorrower = successor.R4[Coll[Byte]].get
        val successorIndex = successor.R5[Int].get
        val successorThresholdPenalty = successor.R6[Coll[Long]].get
        val successorDexNft = successor.R7[Coll[Byte]].get
        val successorUserPk = successor.R8[GroupElement].get

        // Get values from dexBox
        val dexBox = CONTEXT.dataInputs(1)
        val ergInDexPool = dexBox.value
        val tokenInDexPool = dexBox.tokens(2)._2
        val nanoErgsPerToken = ergInDexPool / tokenInDexPool

        // Validate dexBox
        val validDexBox = dexBox.tokens(0)._1 == currentDexNft

        // Validate successor collateral box
        val validSuccessorScript = successorScript == currentScript
        val retainCollateralType = successorCollateral._1 == currentCollateral._1
        val retainBorrowTokens = successorBorrowTokens == currentBorrowTokens
        val retainRegisters = (
            successorBorrower == currentBorrower &&
            successorIndex == currentIndex &&
            successorThresholdPenalty == currentThresholdPenalty &&
            successorDexNft == currentDexNft &&
            successorUserPk == currentUserPk
        )
        // Check sufficient remaining collateral
        val isCorrectCollateralAmount = successorCollateral._2 >= (loanAmount * liquidationThreshold) / (nanoErgsPerToken * LiquidationThresholdDenomination)

        proveDlog(currentUserPk) && 
        sigmaProp(
            validDexBox &&
            validSuccessorScript &&
            retainCollateralType &&
            retainBorrowTokens &&
            retainRegisters &&			
            isCorrectCollateralAmount
        )
    } else {
        // Output boxes
        val borrowerBox = OUTPUTS(0) // Assumes DEX box will be Output(0) and have tokens defined
        val repaymentBox = OUTPUTS(1)

        // Extract variables from borrowerBox
        val receivedTokens = borrowerBox.tokens(0)

        // Validate repayment
        val validRepaymentScript = blake2b256(repaymentBox.propositionBytes) == RepaymentContractScript
        val validBorrowerScript = borrowerBox.propositionBytes == currentBorrower
        val validRepaymentValue = repaymentBox.value >= totalOwed + MinimumTransactionFee
        val validRecordOfLoan = repaymentBox.tokens(0)._2 == loanAmount && repaymentBox.tokens(0)._1 == currentBorrowTokens._1
        val validBorrowerTokens = receivedTokens == currentCollateral

        // Check repayment conditions
        val repayment = (
          validRepaymentScript &&
          validInterestNFT &&
          validRepaymentValue &&
          validRecordOfLoan &&
          validBorrowerScript &&
          validBorrowerTokens 
        )

		// Check liquidation conditions
        val liquidation = if (INPUTS(0).tokens.size >= 3) {
            val dexBox = INPUTS(0)
            val ergInDexPool = dexBox.value
            val tokenInDexPool = dexBox.tokens(2)._2
            val nanoErgsPerToken = ergInDexPool / tokenInDexPool
            val validDexBox = dexBox.tokens(0)._1 == currentDexNft
            val collateralValue = ((currentCollateral._2 * nanoErgsPerToken * DexLpTax) / (DexLpTaxDenomination)) - MaximumNetworkFee
            val liquidationAllowed = currentCollateral._2 <= (totalOwed * liquidationThreshold) / (LiquidationThresholdDenomination * nanoErgsPerToken) || HEIGHT > ForcedLiquidationHeight
            val borrowerShare = ((collateralValue - totalOwed) * (penaltyDenomination - liquidationPenalty)) / penaltyDenomination
            val applyPenalty = if (borrowerShare < minimumBoxValue) {
                repaymentBox.value >= collateralValue
            } else {
                val validRepayment = repaymentBox.value >= totalOwed + ((collateralValue- totalOwed) * liquidationPenalty / penaltyDenomination)
                val borrwerBox = OUTPUTS(2)
                val validBorrowerShare = borrowerBox.value >= borrowerShare
                validRepayment && validBorrowerShare
            }            
            (
              validRepaymentScript &&
              validDexBox &&
              applyPenalty &&
              liquidationAllowed &&
              validInterestNFT &&
              validRecordOfLoan
            )
        } else {
            false
        }
    sigmaProp(repayment || liquidation)
    }
}
```
