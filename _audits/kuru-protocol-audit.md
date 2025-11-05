---
layout: default
title: "Kuru Protocol Audit"
date: 2024-07-25
tags: [cantina]
---


# KuruForwarder — Price-Dependent Execution Logic Vulnerability

## Summary
The `executePriceDependent` function in **KuruForwarder** mistakenly uses the **bid price** for both buy and sell validations, instead of using the **ask price for buys**.  
This flaw allows "limit" orders to execute even when prices exceed user-defined thresholds, leading to **unexpected and financially disadvantageous trades**.

## Finding Description

In `executePriceDependent`, the validation logic reads:

```solidity
(uint256 currentBidPrice,) = IOrderBook(req.market).bestBidAsk();
require(
    (req.isBelowPrice && req.price < currentBidPrice) ||
    (!req.isBelowPrice && req.price > currentBidPrice),
    PriceDependentRequestFailed(currentBidPrice, req.price)
);
```

### This logic is erroneous because:

* **Buy orders** (`isBelowPrice = true`) should validate against the **ask (lowest seller price)**, not the bid (highest buyer price).

### The correct logic must be:

* **a. Buy:** ask ≤ req.price
* **b. Sell:** bid ≥ req.price

### Why this breaks protocol guarantees

Users expect their price-dependent requests to execute only within their limit prices.
The present bug allows trades at **worse prices**, violating that guarantee and compromising user trust.

## Impact Explanation

**Severity:** High

* **Financial Loss:** Users might unintentionally pay higher prices when buying or receive less when selling, directly impacting capital.
* **Trust Erosion:** Violates basic trading expectations; users can lose confidence in the system.
* **Predictable Abuse:** Given normal spread conditions, attackers or MEV actors can reliably trigger unfavorable executions.

## Likelihood Explanation

**Likelihood:** High

* The faulty logic applies to every `executePriceDependent` call.
* No special conditions or edge cases are required; regular limit-type requests are impacted.
* Any market with a bid–ask spread where ask > bid > userLimit enables exploitation.

## Math: What's Wrong vs. Correct

| **Scenario** | **Bid Price** | **Ask Price** | **User Limit** | **Incorrect Check** | **Result (Bug)** | **Correct Check** | **Expected Behavior** |
|--------------|---------------|---------------|----------------|---------------------|------------------|-------------------|------------------------|
| Buy if price < L | 2010 | 2020 | 2000 | 2000 < 2010 (true) | Executes | 2020 ≤ 2000 (false) | Should not execute |
| Sell if price > L | 2010 | 2020 | 2100 | 2100 > 2010 (true) | Executes | 2010 ≥ 2100 (false) | Should not execute |

In both buy and sell cases, using the **bid** for validation leads to **false positives**.

## Proof of Concept

**Test Case:** test/PriceDependentBug.t.sol

> It is a runnable PoC. You can verify the test case results below.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {Test, console2 as console} from "forge-std/Test.sol";
import {KuruForwarder} from "../contracts/KuruForwarder.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

/// @notice Minimal mock orderbook used only for this PoC
contract MockOrderBook {
    uint256 private bid;
    uint256 private ask;

    /// @dev set simulated bid / ask (use same units the contract expects)
    function setPrices(uint256 _bid, uint256 _ask) external {
        bid = _bid;
        ask = _ask;
    }

    /// @notice return best bid and best ask
    function bestBidAsk() external view returns (uint256, uint256) {
        return (bid, ask);
    }

    /// @notice helpers for logging in test
    function bidPrice() external view returns (uint256) {
        return bid;
    }

    function askPrice() external view returns (uint256) {
        return ask;
    }

    /// @dev dummy callable function that accepts the appended `from` address
    /// KuruForwarder calls target with calldata: selector + req.data + req.from
    function noOp( /*from*/) external {
        // intentionally empty
    }
}

contract PriceDependentBugTest is Test {
    KuruForwarder forwarder;
    MockOrderBook orderBook;

    uint256 userPrivateKey = 0x123456;
    address user;

    function setUp() public {
        // Deploy mock orderbook
        orderBook = new MockOrderBook();

        // Prepare allowed selectors array (length 1)
        bytes4[1] memory selectors;
        selectors[0] = MockOrderBook.noOp.selector;

        // Deploy forwarder implementation and proxy, initialize with selectors
        KuruForwarder implementation = new KuruForwarder();
        bytes memory initData = abi.encodeWithSelector(
            KuruForwarder.initialize.selector,
            address(this), // owner
            selectors
        );
        ERC1967Proxy proxy = new ERC1967Proxy(address(implementation), initData);
        forwarder = KuruForwarder(address(proxy));

        // test user funded
        user = vm.addr(userPrivateKey);
        vm.deal(user, 1 ether);
    }

    function testBug_executePriceDependent() public {
        // 1) Set book: bid = 2010, ask = 2020 (use 1e18 units for clarity)
        orderBook.setPrices(2010e18, 2020e18);

        // 2) Build PriceDependentRequest: user wants to BUY only if price < 2000
        KuruForwarder.PriceDependentRequest memory req = KuruForwarder.PriceDependentRequest({
            from: user,
            market: address(orderBook),
            price: 2000e18,
            value: 0,
            nonce: 1,
            deadline: block.timestamp + 1000,
            isBelowPrice: true, // buy when below price
            selector: MockOrderBook.noOp.selector,
            data: "" // no extra data; forwarder will append req.from
        });

        // 3) Sign the typed-data digest with vm.sign (EIP-712 digest computed below)
        bytes32 digest = _getDigest(req);
        (uint8 v, bytes32 r, bytes32 s) = vm.sign(userPrivateKey, digest);
        bytes memory signature = abi.encodePacked(r, s, v);

        // Logging
        console.log("=== BUG DEMO ===");
        console.log("bid:", orderBook.bidPrice() / 1e18);
        console.log("ask:", orderBook.askPrice() / 1e18);
        console.log("user limit:", req.price / 1e18);
        console.log("isBelowPrice:", req.isBelowPrice ? 1 : 0);

        // 4) Execute on the forwarder (this uses the buggy logic in the version you provided)
        forwarder.executePriceDependent(req, signature);

        // 5) Check the mismatch: current buggy check uses bid; correct check uses ask
        (uint256 bid, uint256 ask) = orderBook.bestBidAsk();

        // forwarder current logic (bug): req.price < bid  => executes
        bool executedWithBug = (req.isBelowPrice && req.price < bid);

        // expected (correct) logic: ask <= req.price  => should execute only if ask <= price
        // Here we test shouldExecute == false (ask > user limit)
        bool shouldExecute = (req.isBelowPrice && ask <= req.price);

        // Assert: the buggy logic evaluates to true but the correct logic is false.
        assertTrue(executedWithBug && !shouldExecute, "BUG NOT REPRODUCED: expected buggy execution but it didn't happen");
    }

    /// @dev Build the EIP-712 digest to match the forwarder's verifyPriceDependent encoding
    function _getDigest(KuruForwarder.PriceDependentRequest memory req) internal view returns (bytes32) {
        bytes32 typeHash = keccak256(
            "PriceDependentRequest(address from,address market,uint256 price,uint256 value,uint256 nonce,uint256 deadline,bool isBelowPrice,bytes4 selector,bytes data)"
        );

        bytes32 dataHash = keccak256(abi.encode(
            typeHash,
            req.from,
            req.market,
            req.price,
            req.value,
            req.nonce,
            req.deadline,
            req.isBelowPrice,
            req.selector,
            keccak256(req.data)
        ));

        bytes32 domainSeparator = keccak256(abi.encode(
            keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)"),
            keccak256(bytes("KuruForwarder")),
            keccak256(bytes("1.0.0")),
            block.chainid,
            address(forwarder)
        ));

        return keccak256(abi.encodePacked("\x19\x01", domainSeparator, dataHash));
    }
}
```

## Test Case Results

```bash
nithin@ScateR:~/SCATERLABs/Auditing/kuru-contracts$ forge test --match-test testBug_executePriceDependent -vvvv
[⠊] Compiling...
[⠊] Compiling 1 files with Solc 0.8.29
[⠒] Solc 0.8.29 finished in 2.91s
Compiler run successful!

Ran 1 test for test/PriceDependentBug.t.sol:PriceDependentBugTest
[PASS] testBug_executePriceDependent() (gas: 112683)
Logs:
  === BUG DEMO ===
  bid: 2010
  ask: 2020
  user limit: 2000
  isBelowPrice: 1

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.28ms (1.27ms CPU time)
```

**Result:**
The test confirms: execution went through even though ask (2020) exceeds user limit (2000).
The runnable Foundry test `testBug_executePriceDependent()` successfully reproduces this behavior.

## Recommendation

Replace the flawed condition with:

```solidity
(uint256 bid, uint256 ask) = IOrderBook(req.market).bestBidAsk();

require(
    (req.isBelowPrice && ask <= req.price) ||
    (!req.isBelowPrice && bid >= req.price),
    PriceDependentRequestFailed(req.isBelowPrice ? ask : bid, req.price)
);
```

### Why This Fix Works

* Buys only execute when **ask ≤ user limit** — user never pays more than intended.
* Sells only execute when **bid ≥ user limit** — user never sells for less than intended.
* Maintains logical correctness for both order directions.

⚠️ **Immediate Action Required:**
Fix this **logic vulnerability** to prevent unintended executions and protect user funds.

