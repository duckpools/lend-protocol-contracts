```scala
{
	val timePassed = OUTPUTS(0).R4[Long].get - SELF.R4[Long].get

	SELF.propositionBytes == OUTPUTS(0).propositionBytes &&
	SELF.value == OUTPUTS(0).value &&
	OUTPUTS(0).R4[Long].get <= HEIGHT &&
	OUTPUTS(0).R4[Long].get > SELF.R4[Long].get &&
	OUTPUTS(0).tokens(0)._1 == SELF.tokens(0)._1 &&
	OUTPUTS(0).tokens(0)._2 == SELF.tokens(0)._2 - timePassed * 10000000 // allow 10000000 token withdraw per block
}
```
