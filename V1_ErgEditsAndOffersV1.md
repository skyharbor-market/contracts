Erg Sales Contract Version 1.1

Title: Erg Edits V1 Contract with offers feature

Description: Building on ErgV1 contract, allows for price editing and offers.

P2S: 26tpZU6i6zeBRuzZVo7tr47zjArAxkHP8x7ijiRbbrvLhySvSU84bH8wNPGGG27EwhJfJLe7bvRPv6B1jQFQqrWUQqBX1XJpVGoiNgveance6JFZ4mKv1KkRE8nBSB3jKBGnVJjJF6wR1Z8YXRsUqrTff4bfmtbhaRRjibnDDtKhS71spfjjTBeU1AhhQpitCDg4NFxmTLyV1arE7G2riZKzDryjWnCiEJGzWNxYtVt8uDxd3qNSRE5sHECwcsb98x7rn4q4FyHMvvWrRMPFfVgAQd5wHCAHwhMEdqUrSFQVkmUMavju8CLAgCNcVFjUBKPX4ooEHLUw3QkxS9Jp6fAFAGmzJ6QVD71mAZYMYhoEQnFyUBx1txJjVJjCrcZsW43dimbt5su4ahATJ8qRtWgwat8vTViTVXAcBmUSnqbqhAqTCxcsS5EFS6ApJSfthPHYUyXwtcbTptfdnUx1e5hEiGcwxoQ8ivufNNiZE9xkxi4nsBBrBVBJ7pfSSoHvbodkzLrq91RHYrvuatyLuBSxgJxs198xUQhULqxmWwgthJLrG5VVfVYH


Script:
```scala 
//// LIST NFT GUARD SCRIPT ////

/* //// Register Details ////
SELF.R4(Long) : Sale price in nanoErg
SELF.R5(Raw Address) : Seller Receive Address
SELF.R6(Box) : NFT issuer box
SELF.R7(GroupElement) : Public Key of Seller
OUTPUTS(0).R4(Coll[Bytes]) : Sale Identifier */

{
	val offersContract = fromBase58("AbEGezSUoqbmxonxbP9o8Nx7r88XsctH22fPMhg9R1Tr")
	if (OUTPUTS.size > 3 && blake2b256(INPUTS(0).propositionBytes) != offersContract) { 
		// Purchase Conditions //
		
		val salePrice     = SELF.R4[Long].get
		val royaltyBox    = SELF.R6[Box].get
		val serviceReceives   = salePrice / 50
		
		val purchaseNFT = if (royaltyBox.R4[Int].isDefined) {
			// Royalty Applied //
			
			val royaltyPercentage = royaltyBox.R4[Int].get
			
			val royaltyReceives = SELF.R4[Long].get * royaltyPercentage / 1000        // royaltyPercentage is royalty percentage * 1000
			val sellerReceives  = SELF.R4[Long].get - serviceReceives - royaltyReceives
			
			val sellerBox  = OUTPUTS(0)
			val serviceBox = OUTPUTS(1)
			
			allOf(
				Coll(
					sellerBox.value >= sellerReceives, 
					sellerBox.propositionBytes == SELF.R5[Coll[Byte]].get, // Seller Address
					sellerBox.R4[Coll[Byte]].get == SELF.id, // Unique Identifier to prevent same-price attack
					serviceBox.value >= serviceReceives,
					serviceBox.propositionBytes == fromBase58("1sw5t6iJRxzSjNGvSRw8kcaTADxEG52wGMVgSjdXPLMbhvUM"), // Service Address
					if (royaltyReceives != 0 && royaltyPercentage > 0 && royaltyPercentage < 900) {
						// pay royalty if non-zero
						OUTPUTS(2).value >= royaltyReceives && OUTPUTS(2).propositionBytes == royaltyBox.propositionBytes
						} else{true}, 
					royaltyBox.id == SELF.tokens(0)._1 // Forces royalty to be paid to issuer box address
				)
			) 
		} else {
			// Non-royalty sale (royaltyBox.R4 not defined)
			val sellerReceives = salePrice - serviceReceives
			allOf(
				Coll(
					OUTPUTS(0).value >= sellerReceives, 
					OUTPUTS(0).propositionBytes == SELF.R5[Coll[Byte]].get, // Seller Address
					OUTPUTS(0).R4[Coll[Byte]].get == SELF.id, // Unique Identifier to prevent same-price attack
					OUTPUTS(1).value >= serviceReceives, 
					OUTPUTS(1).propositionBytes == fromBase58("1sw5t6iJRxzSjNGvSRw8kcaTADxEG52wGMVgSjdXPLMbhvUM"), // Service Address
					royaltyBox.id == SELF.tokens(0)._1 // Forces royalty to be paid to issuer
				)
			) 
		}
		sigmaProp(purchaseNFT) 
		} else {
			// Refund Condition (OUTPUT size <= 3)
			val pubKey = SELF.R7[GroupElement].get
			proveDlog(pubKey)
		}
}

```
