```scala
{
    // Constants
    val RepaymentContractScript = fromBase58("GNPRjrvm3YH2pGFuLNXr4KfW4gjU1RL1D5hWsCSvpA4S")
    val InterestBoxNft = fromBase58("7vkbEYNKCdtBAQy62YdoegYzNTAykVAcfvToNwz65FhZ")
    val InterestContractDenomination = 1000000L
    val MaximumNetworkFee = 4000000
    val DexLpTax = 993 // 99.3% retained
    val DexLpTaxDenomination = 1000 
    val LiquidationThresholdDenomination = 10
    val penaltyDenomination = 10
    val MinimumTransactionFee = 1000000L
    val ForcedLiquidationHeight = 2000000
    val minimumBoxValue = 1000000

    // Output boxes
    val borrowerBox = OUTPUTS(0) // Assumes DEX box will be Output(0) and have tokens defined
    val repaymentBox = OUTPUTS(1)

    // Data input
    val interestBox = CONTEXT.dataInputs(0)

    // Extract variables from SELF
    val heldTokens = SELF.tokens(0)
    val heldBorrowTokens = SELF.tokens(1)
    val loanAmount = heldBorrowTokens._2
    val borrower = SELF.R5[Coll[Byte]].get
    val interestIndex = SELF.R6[Int].get
    val liquidationThreshold = SELF.R7[Coll[Long]].get(0)
    val liquidationPenalty = SELF.R7[Coll[Long]].get(1)
    val collateralDexNft = SELF.R8[Coll[Byte]].get

    // Extract variables from borrowerBox
    val receivedTokens = borrowerBox.tokens(0)

    // Calculate interest
    val historicalRates = interestBox.R4[Coll[Long]].get
    val historySize = historicalRates.size
    val historyToScan = historicalRates.slice(interestIndex, historySize)
    val compoundedInterest = historyToScan.fold(InterestContractDenomination, {(z:Long, base:Long) => z + base - InterestContractDenomination})

    // Validate repayment
    val validRepaymentScript = blake2b256(repaymentBox.propositionBytes) == RepaymentContractScript
    val validBorrowerScript = borrowerBox.propositionBytes == borrower
    val validInterestNFT = interestBox.tokens(0)._1 == InterestBoxNft
    val validRepaymentValue = repaymentBox.value >= (loanAmount * compoundedInterest / InterestContractDenomination) + MinimumTransactionFee
    val validRecordOfLoan = repaymentBox.tokens(0)._2 == loanAmount && repaymentBox.tokens(0)._1 == heldBorrowTokens._1
    val validBorrowerTokens = receivedTokens == heldTokens

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
        val totalOwed = loanAmount * compoundedInterest / InterestContractDenomination
        val validDexBox = dexBox.tokens(0)._1 == collateralDexNft
        val collateralValue = ((heldTokens._2 * nanoErgsPerToken * DexLpTax) / (DexLpTaxDenomination)) - MaximumNetworkFee
        val liquidationAllowed = heldTokens._2 <= (totalOwed * liquidationThreshold) / (LiquidationThresholdDenomination * nanoErgsPerToken) || HEIGHT > ForcedLiquidationHeight
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
    // Evaluate and return the final condition
    sigmaProp(repayment || liquidation)
}
```
