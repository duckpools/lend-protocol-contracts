```scala
{
{
    // Constants
    val RepaymentContractScript = fromBase58("GvFgSpew3VuFc7Tyy4eFmLdyuPqVNDyx7ZTqWwiJUjYy")
    val InterestBoxNft = fromBase58("7vkbEYNKCdtBAQy62YdoegYzNTAykVAcfvToNwz65FhZ")
    val InterestMultiplier = 1000000L
    val MaximumNetworkFee = 4000000
    val DexLpTax = 993 // 99.3% retained
    val DexLpTaxDenomination = 1000 
    val LiquidationThresholdDenomination = 10
    val MinimumTransactionFee = 1000000L
    val ForcedLiquidationHeight = 2000000

    // Output boxes
    val borrowerBox = OUTPUTS(0) // Assumes DEX box will be Output(0) and have tokens defined
    val repaymentBox = OUTPUTS(1)

    // Data input
    val interestBox = CONTEXT.dataInputs(0)

    // Extract variables from SELF
    val heldTokens = SELF.tokens(0)
    val loanAmount = SELF.R4[Long].get
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
    val compoundedInterest = historyToScan.fold(InterestMultiplier, {(z:Long, base:Long) => z + base - InterestMultiplier})

    // Validate repayment
    val validRepaymentScript = blake2b256(repaymentBox.propositionBytes) == RepaymentContractScript
    val validBorrowerScript = borrowerBox.propositionBytes == borrower
    val validInterestNFT = interestBox.tokens(0)._1 == InterestBoxNft
    val validRepaymentValue = repaymentBox.value >= (loanAmount * compoundedInterest / InterestMultiplier) + MinimumTransactionFee
    val validRecordOfLoan = repaymentBox.R4[Long].get == loanAmount
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
        val totalOwed = loanAmount * compoundedInterest / InterestMultiplier
        val validDexBox = dexBox.tokens(0)._1 == collateralDexNft
        val liquidationAllowed = heldTokens._2 <= (totalOwed * liquidationThreshold) / (LiquidationThresholdDenomination * nanoErgsPerToken) || HEIGHT > ForcedLiquidationHeight
        val minValueMet = repaymentBox.value >= ((heldTokens._2 * nanoErgsPerToken * DexLpTax) / (DexLpTaxDenomination)) - MaximumNetworkFee
        (
          validRepaymentScript &&
          validDexBox &&
          minValueMet &&
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
