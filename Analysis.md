# Analysis

- [LoopringProtocol.sol](#loopringprotocolsol)
  * [Description of contract system](#description-of-contract-system)
  * [Functions](#functions)
- [LoopringProtocolImpl.sol](#loopringprotocolimplsol)
  * [Description of contract system](#description-of-contract-system-1)
  * [Functions](#functions-1)
    + [submitRing](#submitring)
    + [cancelOrder](#cancelorder)
    + [setCutoff](#setCutoff)
    + [verifyRingHasNoSubRing](#verifyringhasnosubring)
    + [verifyTokensRegistered](#verifytokensregistered)
    + [handleRing](#handlering)
    + [createTransferBatch](#createtransferbatch)
    + [settleRing](#settlering)
    + [verifyMinerSuppliedFillRates](#verifyminersuppliedfillrates)
    + [calculateRingFees](#calculateringfees)
    + [calculateRingFillAmount](#calculateringfillamount)
    + [calculateOrderFillAmount](#calculateorderfillamount)
    + [scaleRingBasedOnHistoricalRecords](#scaleringbasedonhistoricalrecords)
    + [getSpendable](#getspendable)
    + [verifyInputDataIntegrity](#verifyinputdataintegrity)
    + [assembleOrders](#assembleorders)
    + [validateOrder](#validateorder)
    + [calculateOrderHash](#calculateorderhash)
    + [verifySignature](#verifysignature)
- [TokenTransferDelegate.sol](#tokentransferdelegatesol)
  * [Description of contract system](#description-of-contract-system-2)
  * [Modifiers](#modifiers)
  * [onlyAuthorized()](#onlyauthorized--)
  * [Functions](#functions-2)
    + [authorizeAddress](#authorizeaddress)
    + [isAddressAuthorized](#isaddressauthorized)
    + [transferToken](#transfertoken)
    + [batchTransferToken](#batchtransfertoken)
- [TokenRegistry.sol](#okenregistrysol)
  * [Description of contract system:](#description-of-contract-system-)
  * [Functions](#functions-3)
    + [registerToken](#registertoken)
    + [unregisterToken](#unregistertoken)
    + [isTokenRegisteredBySymbol](#istokenregisteredbysymbol)
    + [isTokenRegistered](#istokenregistered)
    + [getAddressBySymbol](#getaddressbysymbol)
- [RinghashRegistry.sol](#ringhashregistrysol)
  * [Description of contract system](#description-of-contract-system-3)
  * [Functions](#functions-4)
    + [submitRinghash](#submitringhash)
    + [batchSubmitRinghash](#batchsubmitringhash)
    + [calculateRinghash](#calculateringhash)
    + [computeAndGetRinghashInfo](#computeandgetringhashinfo)
    + [canSubmit](#cansubmit)
    + [isReserved](#isreserved)
- [Library Contracts](#library-contracts)

# LoopringProtocol.sol
## Description of contract system
This contract is an interface for LoopringProtocolImpl.sol

# LoopringProtocolImpl.sol

## Description of contract system
Description of contract system: This contract implements the functionality of the Loopring Protocol as outlined in the whitepaper. Methods for creating, assembling, and submitting ring orders are implemented as outlined in the LoopringProtocol.sol interface. Various calculations and checks are completed during each step of the process by private helper functions.
## Functions
### submitRing
``` 
submitRing(
        address[2][]  addressList,
        uint[7][]     uintArgsList,
        uint8[2][]    uint8ArgsList,
        bool[]        buyNoMoreThanAmountBList,
        uint8[]       vList,
        bytes32[]     rList,
        bytes32[]     sList,
        address       ringminer,
        address       feeRecipient)
```

This function submits an order-ring for validation and settlement. First, a ringIndex mutex is checked to prevent ring re-submission, the ring size is checked for validity, the data input parameters are checked for integrity with a call to the verifyInputDataIntegrity() function, and the token addresses are checked for registry with a call to the verifyTokensRegistered() function. Next, a call to the RinghashRegistry contract’s function computeAndGetRinghashInfo() is made to obtain the ringhash and ring attributes for the order ring. A function call to verifySignature() looks at the last entry in the v, r, s signature arrays to verify the hash of the computed ring. After, a call to the assembleOrders() function assembles input data into Order structs so they can be passed to other functions. The miner is set as the fee recipient.  Lastly, the function handleRing() is called which processes some final checks on the ring and then proceeds to submit payments for the ring. In addition, the mutex is reset, indicating that a submission has been completed.
### cancelOrder
```
cancelOrder(
        address[3] addresses,
        uint[7]    orderValues,
        bool       buyNoMoreThanAmountB,
        uint8      marginSplitPercentage,
        uint8      v,
        bytes32    r,
        bytes32    s)
```
This function first reconstructs an order given the parameters for an Order struct and verifies that the reconstructed order has a valid hash and signature. Next it adds the Order struct to a mapping of canceled or filled orders using the order hash as the key. Lastly, an OrderCancelled event is fired to indicate that the order has been cancelled.
### setCutoff
```
setCutoff(uint cutoff)
```
This function sets a cutoff timestamp to invalidate all orders whose timestamp is smaller than or equal to the new value of the address's cutoff timestamp.
### verifyRingHasNoSubRing
```
verifyRingHasNoSubRing(
        uint          ringSize,
        OrderState[]  orders)
```

This function ensures that the ring does not have two orders with the same selling token by iterating through all orders in a ring.
### verifyTokensRegistered
```
verifyTokensRegistered(
        uint          ringSize,
        address[2][]  addressList)
```
This function loads an array of length ringSize with the token addresses in addressList. Every address in this array is tested and enforced by requiring the TokenRegistry’s areAllTokensRegistered() function to return true.  
### handleRing
```
handleRing(
        TokenTransferDelegate delegate,
        uint          ringSize,
        bytes32       ringhash,
        OrderState[]  orders,
        address       miner,
        address       feeRecipient,
        bool          isRinghashReserved)
```
This function makes final checks before payments to orders in an order ring. It is called by the submitRing() function. First, calls to verifyRingHasNoSubRing() and verifyMinerSuppliedFillRates() are made to validate inputs and exchange rates. Next, orders are scaled down by calling scaleRingBasedOnHistoricalRecords() and calculateRingFillAmount() which determine the maximum flow of value possible for the entire ring. After, ring fees are calculated by a call to calculateRingFees(). Lastly, the function settleRing() is called to process payments and a RingMined event is fired.
### createTransferBatch
```
createTransferBatch(
        uint          ringSize,
        OrderState[]  orders)
```
This function uses the ringSize and list of orders to create a batch (bytes32 array) with all the relevant information and format needed for TokenTransferDelegate’s batchTransferToken(). 
### settleRing
```
settleRing(
        TokenTransferDelegate delegate,
        uint          ringSize,
        OrderState[]  orders,
        bytes32       ringhash,
        address       feeRecipient,
        address       _lrcTokenAddress)
```
This function iterates through all orders in a ring, adds them to the canceledOrFilled mapping, and fires an OrderFilled event for each. Then, the createTransferBatch() method is called to assemble a batch order. After, a call to the TokenTransferDelagate.sol contract’s batchTransferToken() method is made under the specified token transfer delegate to submit the orders and pay fees to the ring miner.
###  verifyMinerSuppliedFillRates
```
verifyMinerSuppliedFillRates(
        uint          ringSize,
        OrderState[]  orders)
```
This function checks iterates through all the orders a miner provides and calculates the rate ratio of the orders. This rate ratio is then checked to enforce that it is less than or equal to the rateRatioCVSThreshold constant that the Loopring team determined.
### calculateRingFees
```
calculateRingFees(
        TokenTransferDelegate delegate,
        uint            ringSize,
        OrderState[]    orders,
        address         feeRecipient,
        address         _lrcTokenAddress)
```
This function calculates the fees for submitting a ring based on a miner’s choice to take LRC or take the margin split.
### calculateRingFillAmount
```
calculateRingFillAmount(
        uint          ringSize,
        OrderState[]  orders)
```
This function iterates through the list of orders and calls the calculateOrderFillAmount() function on every iteration to determine the limiting factor of the list of orders (similar to a max-flow algorithm). The second for-loop updates all the orders  in the order book up to the limiting factor order with the minimum order filled to accurately reflect a correct flow between all orders.
### calculateOrderFillAmount
```
calculateOrderFillAmount(
        OrderState        state,
        OrderState        next,
        uint              i,
        uint              j,
        uint              smallestIdx)
```
This function returns the smallest order's index given two OrderState’s and the current smallest index. This function is called multiple times by calculateRingFillAmount()
### scaleRingBasedOnHistoricalRecords
```
scaleRingBasedOnHistoricalRecords(
        uint          ringSize,
        OrderState[]  orders)
```
This function iterates through a list of orders and scales down all orders based on historical fill or cancellation stats but keeps the order's original exchange rate.
### getSpendable
```
getSpendable(
        TokenTransferDelegate delegate,
        address tokenAddress,
        address tokenOwner)
```
This function returns the amount of ERC20 token that can be spent by this contract.
### verifyInputDataIntegrity
```
verifyInputDataIntegrity(
        uint          ringSize,
        address[2][]  addressList,
        uint[7][]     uintArgsList,
        uint8[2][]    uint8ArgsList,
        bool[]        buyNoMoreThanAmountBList,
        uint8[]       vList,
        bytes32[]     rList,
        bytes32[]     sList)
```
This function verifies the input data of an order ring. The length of the order ring is checked against the length of all order data arrays to ensure that there are no missing or extra orders. In addition, all ring-mining related arguments are iterated through and validated.
### assembleOrders
```
assembleOrders(
        TokenTransferDelegate delegate,
        address[2][]    addressList,
        uint[7][]       uintArgsList,
        uint8[2][]      uint8ArgsList,
        bool[]          buyNoMoreThanAmountBList,
        uint8[]         vList,
        bytes32[]       rList,
        bytes32[]       sList)
```
This function takes in parameters for a set of orders as input, assembles a corresponding “Order” struct, calls calculateOrderHash(), verifySignature(), validateOrder(), and populates an array of OrderState structs for each order. In addition, it checks that each order has a nonzero available amount of tokens.
### validateOrder
```
validateOrder(
        Order        order,
        uint         timestamp,
        uint         ttl,
        uint         salt)
```
This function checks that the Order’s addresses, amounts, ttl, and salt are nonzero; that the timestamp is greater than the specified cutoff, less than the current block number, and within the specified ttl range; and that the Order’s “marginSplitPercentage” is less than or equal to the base percentage. This function reverts state should any of the checks fail.
### calculateOrderHash
```
calculateOrderHash(
        Order        order,
        uint         timestamp,
        uint         ttl,
        uint         salt)
```
This function computes a hash on the data contained by an “Order” struct and the other specified inputs.
### verifySignature
```
verifySignature(
        address signer,
        bytes32 hash,
        uint8   v,
        bytes32 r,
        bytes32 s)
```
This function checks that the “signer” matches the address that computed a valid signature on the “hash” and reverts state if the addresses do not match.
# TokenTransferDelegate.sol
## Description of contract system
The TokenTransferDelegate is interfaced with in the implementation of the Loopring Protocol. It serves as a decentralized, trustless intermediary such that orders can be placed asynchronously, without having to know who the next participant in the ring would be. 
## Modifiers
## onlyAuthorized()
```onlyAuthorized()```
This modifier intends to verify that the sender of the message is an authorized address. It makes a function call to verify such a statement but could similarly use the one line body of this function to achieve the same result. This may save on gas as well.
## Functions
### authorizeAddress
```
authorizeAddress(address addr)
```
This function adds the address of a LoopringProtocol contract. It can only be called by the owner of the contract.
### isAddressAuthorized
```
isAddressAuthorized(address addr)
```
This function traverses mapping using known AddressInfo structs. This while loop will always fail however, since one condition of the while loop is unsatisfiable with the initial condition. 
getLatestAuthorizedAddresses(uint max)
This function traverses mapping using known AddressInfo structs. It attempts to aggregate the last max number of addresses registered and return them in an array. 
Finding: This while loop will always fail however, since one condition of the while loop is unsatisfiable with the initial condition. Since max > 0 by the definition of a uint the loop will never execute, so this function will always return an empty array.
### transferToken
```
transferToken(address token,
        address from,
        address to,
        uint value)
```
This function is critical to the use of the TokenTransferDelegate. When called, it transfers an amount of some ERC20 token from one party to another using a pre approved allowance. 
### batchTransferToken
```
batchTransferToken(uint ringSize, 
address lrcTokenAddress,
address feeRecipient,
bytes32[] batch)
```
This function takes a batch of orders and sends tokens to their intended recipients through the ring as specified by the batch. Again, this can only be called by the LoopringProtocol contract. The function loops through the orders and executes by transferring tokens as specified by orders in the ring.
# TokenRegistry.sol
## Description of contract system: 
This contract maintains an array of token contract addresses that are able to be used by an exchange using the Loopring protocol. A mapping of token contract addresses to booleans to keep track of registered and unregistered tokens. In addition, a mapping of token symbols to token addresses is stored. The contract’s functions provide the logic to register, unregister, lookup, and return any token that the contract has interacted with. 
## Functions
### registerToken
```
registerToken(address _token, string _symbol)
```
This function pushes a new token to the token array, stores the token symbol, and sets a boolean corresponding with the token address to true if the token has not been registered before or has its boolean set to false.
This function can only be called by the contract owner.
### unregisterToken
```
unregisterToken(address _token, string _symbol)
```
This function removes a token from the token array, deletes the token symbol, and flips the boolean corresponding with the token address to false if the supplied token address and symbol are currently stored. 
This function can only be called by the contract owner.
### isTokenRegisteredBySymbol
```
isTokenRegisteredBySymbol(string symbol)
```
This function looks up a token address corresponding to the symbol parameter and returns the value of comparing the returned address to the zero address.
### isTokenRegistered
```
isTokenRegistered(address _token)
```
This function looks up a token boolean corresponding to the address parameter in the registered tokens mapping and returns it.
areAllTokensRegistered(address[] tokenList)
This function iterates through the array of stored tokens and looks up each token’s corresponding boolean value. If any token’s boolean is false (it is unregistered) the function outputs false. Otherwise, it returns true.
### getAddressBySymbol
```
getAddressBySymbol(string symbol)
```
This function looks up a token address corresponding to the symbol parameter and returns the value of the returned address.
# RinghashRegistry.sol
## Description of contract system
This contract maintains a mapping of valid ringhashes to submissions. The two public functions submitRinghash() and batchSubmitRinghash() implement logic for calculating and registering a new ring or batch of rings and their hashes to the contract.
## Functions
### submitRinghash
```
submitRinghash(
        address     ringminer,
        bytes32     ringhash)	
```
This function stores a submission in the mapping of all submissions at a certain block number and logs an event with the same info.
### batchSubmitRinghash
```
batchSubmitRinghash(
        address[]     ringminerList,
        bytes32[]     ringhashList)	
```
This function submits a ringhash using the submitRinghash() function for every submission pairing in ringminerList and ringHashList.
### calculateRinghash
```
calculateRinghash(
        uint        ringSize,
        uint8[]     vList,
        bytes32[]   rList,
        bytes32[]   sList)
```
This function calculates the ringhash by calling hashing the xorReduce() between all the ECSDA signature parameter lists and the ringSize.
xorReduce() is implemented in the Loopring defined library.
### computeAndGetRinghashInfo
```
computeAndGetRinghashInfo(
        uint        ringSize,
        address     ringminer,
        uint8[]     vList,
        bytes32[]   rList,
        bytes32[]   sList)
```
This function calculates the ringhash with the calculateRingHash() function and returns if it can be submitted and if it is reserved.
### canSubmit
```
canSubmit(
        bytes32 ringhash,
        address ringminer)
```
This function checks if the ringhash can be submitted given a ringhash and ringminer pair. 
### isReserved
```
isReserved(
        bytes32 ringhash,
        address ringminer)
```
This function checks if the ringhash was submitted and still valid.

# Library Contracts
 The following contracts were used:
- ERC20.sol
- MathBytes32.sol
- MathUint.sol
- MathUint8.sol
- Ownable.sol
