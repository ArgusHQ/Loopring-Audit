# Additional Comments
- 20 million LRC was intentionally burned to  [```0x0```](https://etherscan.io/token/0xef68e7c694f40c8202821edf525de3782458639f?a=0x0000000000000000000000000000000000000000) by the Loopring team to reduce circulating volume.
- Math library contracts are implemented by the Loopring team. A ```div()``` function is not implemented, but is inconsequential because manually dividing is safe in current solidity version, 0.4.18. 
- Gas efficiency. Mapping vs array: It is cheaper to initialize an array and add elements to it, when compared to a mapping
