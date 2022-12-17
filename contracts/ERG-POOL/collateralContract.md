```scala
{
    	val repaymentContract     = fromBase58("6YmLa5cHbNo3KogBYtDrLSubdPLRqcEymVA17ZfjNaTz")
	val interestBoxNFT        = fromBase58("Gne6mUgPuJmzATwkNCpMyZp173K76dQEC9jMTGpDvP3p")
	val sigUSDTokenId         = fromBase58("GYATox71P9XAERmzoDdTGELa62f5ALyjxJLRSfJfKsh")
	val sUsdErgDexPoolNFT     = fromBase58("BJbaZAXMoFm9gi2MBXA9eyPi38ugjKZ66SQrnwQmoDNj")
	val sigRsvTokenId         = fromBase58("1uuSELXj5meePwVr5PX1ndCSEsUvqV4EmNunatgho9y")
	val sRsvErgDexPoolNFT     = fromBase58("2ybHuZL1EdKsc84KaKgKsbQn9KiL7Py8PWedunbrc7dx")
	val InterestMultiplier    = 1000000L
	val maxTxFee              = 4000000
	val decimalMultiplier     = 10000000L
	val dexLpFee              = 993 //99.3% retained
	val lpMultiplier          = 1000 
	val liquidationThreshold  = 12
	val liquidationMultiplier = 10
	val minTxFee              = 1000000L
	val forcedLiquidateHeight = 1000000
	
 
    val borrowerBox  = OUTPUTS(0) // No safety branches done, assumes DEX box will be Output(0) and have tokens defined
	val repaymentBox = OUTPUTS(1)   
	val interestBox  = CONTEXT.dataInputs(0)
    
	val heldTokens    = SELF.tokens(0)
	val loanAmount    = SELF.R4[Long].get
	val borrower      = SELF.R5[Coll[Byte]].get
	val interestIndex = SELF.R6[Int].get
	
	val receivedTokens = borrowerBox.tokens(0)
	
	val historicalRates = interestBox.R4[Coll[Long]].get
	val historySize     = historicalRates.size 
	val historyToScan   = historicalRates.slice(interestIndex, historySize)
	
	val compoundedInterest = historyToScan.fold(InterestMultiplier, {(z:Long, base:Long) => z + base - InterestMultiplier})
	
	val validRepaymentScript = blake2b256(repaymentBox.propositionBytes) == repaymentContract
	val validBorrowerScript  = borrowerBox.propositionBytes == borrower
	val validInterestNFT     = interestBox.tokens(0)._1 == interestBoxNFT
	
	val validRepaymentValue = repaymentBox.value >= (loanAmount* compoundedInterest / InterestMultiplier) + minTxFee
	val validRecordOfLoan   = repaymentBox.R4[Long].get == loanAmount
	
	val validBorrowerTokens = receivedTokens == heldTokens
    
    val repayment = (
		validRepaymentScript &&
        validInterestNFT &&
        validRepaymentValue &&
		validRecordOfLoan &&
        validBorrowerScript &&
		validBorrowerTokens
    )
	
	val liquidation = if (INPUTS(0).tokens.size >= 3) {
		val dexBox = INPUTS(0)
			
		val ergInDexPool    = dexBox.value 
		val tokenInDexPool  = dexBox.tokens(2)._2 
		val nanoErgsPerToken = ergInDexPool / tokenInDexPool
		
		val totalOwed = loanAmount * compoundedInterest / InterestMultiplier
			
		val validDexBox = if (heldTokens._1 == sigUSDTokenId) {
			dexBox.tokens(0)._1 == sUsdErgDexPoolNFT
		} else if (heldTokens._1 == sigRsvTokenId) {
			dexBox.tokens(0)._1 == sRsvErgDexPoolNFT
		} else {
			false
		}
		
		val liquidationAllowed        = heldTokens._2 <= (totalOwed * liquidationThreshold) / (liquidationMultiplier * nanoErgsPerToken) || HEIGHT > forcedLiquidateHeight 
		val minValueMet               = repaymentBox.value >= ((heldTokens._2 * nanoErgsPerToken * dexLpFee) / (lpMultiplier)) - maxTxFee
		
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
	
	sigmaProp(repayment || liquidation)
}
