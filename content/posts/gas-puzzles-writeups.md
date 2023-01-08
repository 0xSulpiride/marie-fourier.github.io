---
author: "Marie Fourier"
title: "RareSkills Gas Puzzles"
date: "2023-01-06"
description: "Writeups on gas puzzles by RareSkills - https://github.com/RareSkills/gas-puzzles"
tags: [
    "ctf",
    "writeups"
]
---

Gas Puzzles by [Rareskills](https://github.com/RareSkills) are a sequence of smart contracts to practice gas optimization. These are used as practice assignments for RareSkills.io and the [Udemy Gas Optimization Course](https://www.udemy.com/course/advanced-solidity-understanding-and-optimizing-gas-costs/?referralCode=C4684D6872713525E349)

## Array Sum
Let's start with the easiest one
```sol
contract OptimizedArraySum {
    uint256[] array;

    function setArray(uint256[] memory _array) external {
        require(_array.length <= 10, "too long");
        array = _array;
    }

    function getArraySum() external view returns (uint256) {
        uint256 sum;
        for (uint256 i = 0; i < array.length; i++) {
            sum += array[i];
        }

        return sum;
    }
}

Gas target
    Current gas use:   23399
    The gas target is: 23396
    You are 3 above the target
```

We can refactor the function by deleting `uint256 sum` and `return` statements. We could also create a static variable for storing the length of the dynamic array `array`, but in this case the memory extension costs more than accessing `array.length` because the length of the array is very small

```sol
function getArraySum() external view returns (uint256 sum) {
    for (uint256 i = 0; i < array.length; i++) {
        sum += array[i];
    }
}

Gas target
    Current gas use:   23391
```

## Require
```sol
contract Require {
    uint256 constant COOLDOWN = 1 minutes;
    uint256 lastPurchaseTime;

    function purchaseToken() external payable {
        require(
            msg.value == 0.1 ether &&
                block.timestamp > lastPurchaseTime + COOLDOWN,
            "cannot purchase"
        );
        lastPurchaseTime = block.timestamp;
        // mint the user a token
    }
}

Gas target
    Current gas use:   43392
    The gas target is: 43317
    You are 75 above the target
```

Setting a slot value from zero to non-zero costs 20k gas\
So, one thing we can do here is setting the value of `lastPurchaseTime` to 1, so the slot is already set a non-zero value when `purchaseToken` is called for the first time.\
Setting a slot value from non-zero to non-zero costs still costs 5000 or 2900, depending on whether the slot is cold or warm (whether is was accessed in current transaction before or not)\
In our case, `lastPurchaseTime` is accessed before setting it is set to a new value in the `require` statement, so `SSTORE` should cost 2900 gas
```sol
contract Require {
    uint256 constant COOLDOWN = 1 minutes;
    uint256 lastPurchaseTime = 1;
    ...
}

Gas target
    Current gas use:   26292
```

Gas usage is 17100 less, which makes sense because `20000 - 2900 = 17100`

I wanted to see how much gas I could save by completely rewriting the `purchaseToken` function in Yul
```sol
function purchaseToken() external payable {
    assembly {
        if or(
            iszero(eq(callvalue(), 100000000000000000)), // 0.1 ether
            iszero(gt(
                timestamp(),
                add(sload(lastPurchaseTime.slot), 60) // 1 minute
            ))
        ) {
            // EIP 838 (https://github.com/ethereum/EIPs/issues/838)
            let error := 0x00
            mstore(error, 0x08c379a0) // error selector
            mstore(add(error, 0x20), 0x20) // string offset
            mstore(add(error, 0x40), 0x0f) // length("cannot purchase") = 0x0f bytes
            mstore(add(error, 0x60), 0x63616e6e6f742070757263686173650000000000000000000000000000000000) // "cannot purchase"
            revert(add(error, 0x1c), sub(0x80, 0x1c)) // error selector bytes start at (error + 28bytes)
        }
        sstore(lastPurchaseTime.slot, timestamp())
    }
}

Gas target
    Current gas use:   26215
```

It is 77 gas less

## Distribute
```sol
contract Distribute {
    address[4] public contributors;
    uint256 public createTime;

    constructor(address[4] memory _contributors) payable {
        contributors = _contributors;
        createTime = block.timestamp;
    }

    function distribute() external {
        require(
            block.timestamp > createTime + 1 weeks,
            "cannot distribute yet"
        );

        uint256 amount = address(this).balance / 4;
        payable(contributors[0]).transfer(amount);
        payable(contributors[1]).transfer(amount);
        payable(contributors[2]).transfer(amount);
        payable(contributors[3]).transfer(amount);
    }
}

Gas target
    Current gas use:   71953
    The gas target is: 57044
    You are 14909 above the target
```

First, the state variables `contributors` and `createTime` are never changed after the construction, so we can make them `immutable`\
Immutable variables won't be stored in the storage, and all references to those variables will be replaced by the values assigned to them in the constructor.\
This will save us a lot of gas

Second, instead of storing `createTime` we can calculate and store the release time. This should reduce the gas usage by 2, since we're not using `ADD`

Third, instead of dividing the contract balance by 4 we could just shift it by 2 bits (since 4 = 2^2), shifting costs 2 gas less than dividing

And finally, we can destroy the contract after the distribution is complete by calling `selfdestruct`. This will transfer the contract balance to a given address and will refund some gas
```sol
contract OptimizedDistribute {
    address payable immutable contributors0;
    address payable immutable contributors1;
    address payable immutable contributors2;
    address payable immutable contributors3;
    uint256 public immutable releaseTime;

    constructor(address payable[4] memory _contributors) payable {
        contributors0 = _contributors[0];
        contributors1 = _contributors[1];
        contributors2 = _contributors[2];
        contributors3 = _contributors[3];
        releaseTime = block.timestamp + 1 weeks;
    }

    function distribute() external {
        require(
            block.timestamp > releaseTime,
            "cannot distribute yet"
        );
        uint256 amount = address(this).balance >> 2;
        contributors0.transfer(amount);
        contributors1.transfer(amount);
        contributors2.transfer(amount);
        selfdestruct(contributors3);
    }
}

Gas target
    Current gas use:   57038
```


## Mint 150
```sol
import "@openzeppelin/contracts/token/ERC721/ERC721.sol";

// You may not modify this contract or the openzeppelin contracts
contract NotRareToken is ERC721 {
    mapping(address => bool) private alreadyMinted;

    uint256 private totalSupply;

    constructor() ERC721("NotRareToken", "NRT") {}

    function mint() external {
        totalSupply++;
        _safeMint(msg.sender, totalSupply);
        alreadyMinted[msg.sender] = true;
    }
}

contract Attacker {
    constructor(address victim) {}
}
```

In this challenge the `Attacker` contract must mint 150 NFTs from the `NotRareToken` contract and send them to the EOA of the attacker in a single transaction that consumes no more than 6,029,700 gas.

_Analysis:_

The solution appears to be straightforward. The `mint()` and `transferFrom()` functions can be called repeatedly to mint and transfer an NFT to the EOA. The `mint()` function increments the `totalSupply` by 1 and mints an NFT with an ID of `totalSupply` to `msg.sender` by calling `_safeMint()`.

It should be noted that the `_safeMint()` function may invoke the `onERC721Received()` function on `msg.sender` if it is a contract, which could result in unnecessary gas consumption. However, as the `mint()` function will be called within the constructor of the `Attacker` contract, the `Attacker.code.length` will be equal to 0, causing `_safeMint()` to treat `Attacker` as an EOA.

To transfer an NFT, the ID of the NFT must be known. `totalSupply` is private and not accessible within the contract, but the value of its slot can be fetched via an RPC call and passed to the constructor. In the `NotRareToken` contract, `totalSupply` is stored in slot 7."

test/Mint150.js:63
```js
const offset = await ethers.provider.getStorageAt(victimToken.address, 7) + 1;
const txn = await attackerContract
    .connect(attacker)
    .deploy([ victimToken.address, offset ]);
...
```
OptimizedAttacker:
```sol
    contract Attacker {
        constructor(address victim, uint256 offset) {
            for (uint256 i = 0; i < 150; i++) {
                NotRareToken(victim).mint();
                IERC721(victim).transferFrom(address(this), msg.sender, i + offset);
            }
        }
    }
```

However, the rules do not indicate that we are permitted to modify the test case in this manner. Let's retrieve the offset within the contract

```sol
contract OptimizedAttacker {
    constructor(address victim) {
        unchecked {
            // in the test case all the existing nfts belong to a single wallet.
            // we can fetch the offset by looking at the balance of that wallet
            uint256 offset = IERC721(victim).balanceOf(IERC721(victim).ownerOf(1)) + 1;

            // minted one NFT before going inside the loop to make the balance of the Attacker non-zero.
            // otherwise every iteration of the loop will set the balance from zero to non-zero (which costs 20k gas)
            // and sending the NFT to EOA will set the balance back to zero again.
            // we can save a lot of gas by keeping the balance of the attacker non-zero
            NotRareToken(victim).mint();
            for (uint256 i = 1; i < 150; i++) {
                NotRareToken(victim).mint();
                IERC721(victim).transferFrom(address(this), msg.sender, i + offset);
            }
            IERC721(victim).transferFrom(address(this), msg.sender, offset);
        }
    }
}
```
```js
Mint150
    Gas target
           Current gas use:   5429661
           The gas target is: 6029700
      ✔ The functions MUST meet the expected gas efficiency (6443ms)
    Business logic
      ✔ The attacker MUST mint 150 NFTs in one transaction (5996ms)
```
