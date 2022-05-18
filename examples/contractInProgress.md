//https://stackoverflow.com/questions/67321111/file-import-callback-not-supported
 
 //SPDX-License-Identifier: MIT

//pragma solidity ^0.8.2;

//contract SimpleStorage {
  //uint256 storedData;
	
  //function get() public view returns (uint) {
    //return storedData;
 //}

  //function set(uint x) public {
   // storedData = x;
 // }

 // function double() public {
  //  storedData *= 2;
 // }
//}

//contract MathTest {
//	function multiply(uint a, uint b) public pure returns (uint) {
   // return a*b;
 // }
//}

//With SushiSwap being a fork of UniSwap, the different functionalities of the contract all relate back to UniSwap V2 (The different UniSwap V2 contracts can be found in this Github repo).

//After defining the version of solidity we will be using (UniSwap V2 is built on 0.6.6), we must import a number of interfaces needed to interact with UniSwap V2.



pragma solidity ^0.6.6;

import './UniswapV2Library.sol';
import './interfaces/IUniswapV2Router02.sol';
import './interfaces/IUniswapV2Pair.sol';
import './interfaces/IUniswapV2Factory.sol';
import './interfaces/IERC20.sol';

//With that out of the way, it is time to start writing the contract itself. One of the first steps is to define a number of crucial variables, such as:

//factory: A central hub within the UniSwap ecosystem that provides information about the liquidity pools.
//deadline: The date the trade is due.
//sushiRouter: A central smart contract in the SushiSwap ecosystem that is used to trade in its liquidity pools.

contract Arbitrage {
  address public factory;
  uint constant deadline = 10 days;
  IUniswapV2Router02 public sushiRouter;

//We can now create the constructor, initializing the value of the UniSwap Factory and sushiRouter:
  
  constructor(address _factory, address _sushiRouter) public {
    factory = _factory;  
    sushiRouter = IUniswapV2Router02(_sushiRouter);
  }

  //The Arbitrage Function
//With the basics now being out of the way, we can create the most important part of the contract which is the “startArbitrage” function, outlining the logic for the arbitrage.

//Within the function, it is important to define the:

//token0: The token we are going to borrow (eg. DAI).
//token1: The token we are going to trade token0 for (eg. ETH).
//amount0: The amount we are going to borrow.
//amount1: This will be 0 for now.
//pairAddress: The address of the pair smart contract of UniSwap for the two tokens (the address of the liquidity pool).

   function startArbitrage(
    address token0, 
    address token1, 
    uint amount0, 
    uint amount1
  ) external {
    address pairAddress = IUniswapV2Factory(factory).getPair(token0, token1);
    require(pairAddress != address(0), 'There is no such pool');
    IUniswapV2Pair(pairAddress).swap(
      amount0, 
      amount1, 
      address(this), 
      bytes('not empty')
    );
  }

  //*When the swap method is called, “address(this)” is entered as a parameter. By doing this, we provide the address we want to receive the borrowed tokens (“this” automatically routes to your address).

//Calling the Flash Loan
//Having written the method in charge of the arbitrage logic, it is important to outline the logic for the FlashSwap itself.

//To do so, the following arguments must be defined:

//_sender: The address that triggered the Flash Loan
//_amount0: The amount borrowed
//_amount1: 0
//address: An array of addresses used to complete the trade.
//amountToken: This will be equal to “_amount0”.
//token0: The address of the first token in Uniswap’s liquidity pool.
//token1: The address of the second token in Uniswap’s liquidity pool.
//IERC20 token: A pointer to the token that we are going to sell on SushiSwap.
//amountRequired: The amount of the initial token that must be reimburshed for the //Flash Loan to UniSwap.

  function uniswapV2Call(
    address _sender, 
    uint _amount0, 
    uint _amount1, 
    bytes calldata _data
  ) external {
    address[] memory path = new address[](2);
    uint amountToken = _amount0 == 0 ? _amount1 : _amount0;
    
    address token0 = IUniswapV2Pair(msg.sender).token0();
    address token1 = IUniswapV2Pair(msg.sender).token1();

    require(
      msg.sender == UniswapV2Library.pairFor(factory, token0, token1), 
      'Unauthorized'
    ); 
    require(_amount0 == 0 || _amount1 == 0);

    path[0] = _amount0 == 0 ? token1 : token0;
    path[1] = _amount0 == 0 ? token0 : token1;

    IERC20 token = IERC20(_amount0 == 0 ? token1 : token0);
    
    token.approve(address(sushiRouter), amountToken);

    uint amountRequired = UniswapV2Library.getAmountsIn(
      factory, 
      amountToken, 
      path
    )[0];
    uint amountReceived = sushiRouter.swapExactTokensForTokens(
      amountToken, 
      amountRequired, 
      path, 
      msg.sender, 
      deadline
    )[1];

   // The only thing now remaining is to pay back the amount due to the lender and send the profit to our wallet.

    IERC20 otherToken = IERC20(_amount0 == 0 ? token0 : token1);
    otherToken.transfer(msg.sender, amountRequired); //Reimbursh Loan
    otherToken.transfer(tx.origin, amountReceived - amountRequired); //Keep Profit
  }
}

//https://thecitadelpub.com/uniswap-flashswaps-make-millions-of-dollars-with-your-code-14d7d5f017dd


  
