# Findings

- [General Findings](#general-findings)
    + [Testing](#testing)
    + [Order Struct](#order-struct)
    + [```var``` Usage](#---var----usage)
    + [Potential Gas Optimization](#potential-gas-optimization)
    + [Documentation](#documentation)
    + [Implicit Typing](#implicit-typing)
- [Significant Findings](#significant-findings)
  * [High](#high)
    + [```LoopringProtocolImpl.sol: submitRing()```](#---loopringprotocolimplsol--submitring-----)
    + [```RinghashRegistry.sol: calculateRingHash()```](#---ringhashregistrysol--calculateringhash-----)
    + [```TokenTransferDelegate.sol: getLatestAuthorizedAddresses()```](#---tokentransferdelegatesol--getlatestauthorizedaddresses-----)
  * [Medium](#medium)
    + [```LoopringProtocolImpl.sol: submitRing()```](#---loopringprotocolimplsol--submitring------1)
    + [```LoopringProtocolImpl.sol: verifyInputDataIntegrity()```](#---loopringprotocolimplsol--verifyinputdataintegrity-----)
    + [```RinghashRegistry.sol: canSubmit()```](#---ringhashregistrysol--cansubmit-----)
- [Low](#low)
    + [```LoopringProtocolImpl.sol: verifyRingHasNoSubRing()```](#---loopringprotocolimplsol--verifyringhasnosubring-----)
    + [```TokenTransferDelegate.sol: authorizeAddress() ```](#---tokentransferdelegatesol--authorizeaddress------)
    + [```TokenTransferDelegate.sol: onlyAuthorized()```](#---tokentransferdelegatesol--onlyauthorized-----)
    + [```TokenRegistry.sol: getAddressBySymbol()```](#---tokenregistrysol--getaddressbysymbol-----)
    + [```TokenRegistry.sol: areAllTokensRegistered()```](#---tokenregistrysol--arealltokensregistered-----)
    + [```RinghashRegistry.sol: calculateRinghash()```](#---ringhashregistrysol--calculateringhash-----)

# General Findings 

### Testing 
The Loopring protocol handles nontrivial logic for managing user finance and would benefit from more unit tests. The testing suite features 21 tests, which provide good coverage for completeness of the Loopring protocol, but should include more checks to be sufficient in considering all edge cases.

### Order Struct
The `Order` struct in `LoopringProtocol.sol` does not implement but has documentation for `timestamp`, `ttl`, `salt`, `v`, `r`, and `s`.

### ```var``` Usage
The `var` keyword is used in several places and is not necessary. The following are references we found:
```
TokenTransferDelegate.sol: Line 178
RinghashRegistry.sol: Lines 149, 166
LoopringProtocolImpl.sol: Lines 254, 278, 514-515, 543-545, 585, 621-622, 776-777, 820, 878
```

### Potential Gas Optimization
For child functions that are used only once to validate inputs for a parent function, it is possible to reduce total gas costs for a parent function by moving child function logic into the parent function.

The following is an example of a potential gas optimization for input validation logic:
```Line 438: handleRing()``` is the parent function. ```Line 450: verifyRingHasNoSubRing()``` is the child function. Making a call to ```verifyRingHasNoSubRing()``` and then computing input validation is more expensive than having the input validation within `handleRing()`.

### Documentation
Both `Rate` and `Order` structs contain `amountS` and `amountB` for different purposes and the use case of these amount values in `Rate` should be documented.

Minor typo concerns were found in codebase documentation.

### Implicit Typing
An ERC20 address is implicitly configured as an ERC20 address and type inference is not necessary. LoopringProtocolImpl.sol Line: 820 TokenDelegateTransfer.sol Lines: 178, 192 Some comments are misspelled. LoopringProtocolImpl.sol Lines: 577, 858

### Gas Limit
There is a hard upper bound, based on the gas limit, on the number of tokens one can add to the token registry and subsequently remove.

# Significant Findings

## Severity explanation:
- High: Affects functionality of protocol 
- Medium: Affects intended functionality, but does not break protocol
- Low: Design discrepancies, optimization opportunities, minor issues

## High

### ```RinghashRegistry.sol: calculateRingHash()```
Line 94: The ring hash is calculated -  using the ECDSA signature parameter lists - as part of the “commitment” phase in the process of submitting an order. However, because the fee recipient is not included in the hash, we are able to initiate a race condition to exploit submitRing() (LoopringProtocolImpl.sol: Line 219). 

**Description of the Exploit**:

Wait for someone to send a “submitRing” transaction
Send a “submitRing” transaction with the same parameters except a fee recipient of your choice. 
You can hope that your transaction is included in a block before the first one, spend more gas, or collude with miners.

**Recommendation**:

Exchanges can ensure that their transactions are included with high enough gas fees or confirming transactions.
Include fee recipient in the ring hash.

## Medium

### ```LoopringProtocolImpl.sol: submitRing()```
Line 258: There should be an additional check `ringminer` is not address(0)

### ```LoopringProtocolImpl.sol: verifyInputDataIntegrity()```

Line 843: There should be an additional check on the elements of the addressList argument. Specifically the function should check the two addresses are not address(0).

### ```RinghashRegistry.sol: canSubmit()```
Lines 151-153: Function logic is unclear. Anyone can submit a ringhash with a ringminer of 0, which will pass the require statement and push events that are false. Change the first check with: ```submission.ringminer == address(0) && submission.block == 0```

# Low
### ```LoopringProtocolImpl.sol: verifyRingHasNoSubRing()```
Lines 411-415: The performance can be improved by checking when tokenS appears in the ring more than once. In addition, there will be an upper bound on the maximum number of times one can run a nested for loop, thus one should limit the maximum ring size to prevent the function from costing too much gas.

### ```TokenTransferDelegate.sol: authorizeAddress() ```
Line 94: This line can be removed as it is repeated on line 100. Only line 100 is neccessary.

### ```TokenTransferDelegate.sol: onlyAuthorized()```
Lines 56-61 and 119-125: This modifier goes through more logic than is needed. It can be replaced with require`(addressInfos[msg.sender].authorized)`.

### ```TokenRegistry.sol: getAddressBySymbol()```
Line 94: The function modifier constant is deprecated and should be view. 

### ```TokenRegistry.sol: areAllTokensRegistered()```
Lines 84-89: This function is called every time submitRing() is invoked and loops through all tokens to ensure they are verified. This becomes more expensive with every token added to the registry and every time an order is submitted. In the event that there exists too many tokens, submitRing() can reach the gas limit before running to completion.

### ```RinghashRegistry.sol: calculateRinghash()```
Line 94: The function is “public” which will leak information if people use it to compute their ring hashes, should they call it in a transaction. 
