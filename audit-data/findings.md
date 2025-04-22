## Info

### [I-1] Centralization Risk

Contracts have owners with privileged rights to perform admin tasks and need to be trusted to not perform malicious updates or drain funds.

<details><summary>1 Found Instances</summary>


- Found in src/WinningToken.sol [Line: 40](src/WinningToken.sol#L40)

    ```solidity
        function mint(address to, uint256 amount) external onlyOwner {
    ```

</details>

### [I-2] Public Function Not Used Internally

If a function is marked public but is not used internally, consider marking it as `external`.

<details><summary>1 Found Instances</summary>


- Found in src/RockPaperScissors.sol [Line: 485](src/RockPaperScissors.sol#L485)

    ```solidity
        function tokenOwner() public view returns (address) {
    ```

</details>

### [I-3] Literal Instead of Constant

Define and use `constant` variables instead of using literals. If the same constant literal value is used multiple times, create a constant state variable and reference it throughout the contract.

<details><summary>4 Found Instances</summary>


- Found in src/RockPaperScissors.sol [Line: 133](src/RockPaperScissors.sol#L133)

    ```solidity
                _timeoutInterval >= 5 minutes,
    ```

- Found in src/RockPaperScissors.sol [Line: 171](src/RockPaperScissors.sol#L171)

    ```solidity
                _timeoutInterval >= 5 minutes,
    ```

- Found in src/RockPaperScissors.sol [Line: 597](src/RockPaperScissors.sol#L597)

    ```solidity
                uint256 fee = (totalPot * PROTOCOL_FEE_PERCENT) / 100;
    ```

- Found in src/RockPaperScissors.sol [Line: 645](src/RockPaperScissors.sol#L645)

    ```solidity
                uint256 fee = (totalPot * PROTOCOL_FEE_PERCENT) / 100;
    ```

</details>

### [I-4] State Change Without Event

There are state variable changes in this function but no event is emitted. Consider emitting an event to enable offchain indexers to track the changes.

<details><summary>1 Found Instances</summary>


- Found in src/RockPaperScissors.sol [Line: 493](src/RockPaperScissors.sol#L493)

    ```solidity
        function setAdmin(address _newAdmin) external {
    ```

</details>


## Low

### [L-1] `GameState.Revealed` is not used in `RockPaperScissors` contract

**Description:** The `GameState.Revealed` state is not used in the `RockPaperScissors` contract. This state is defined in the `GameState` enum but is never assigned or checked in any of the contract's functions.
```javascript
    enum GameState {
        Created,
        Committed,
        Revealed,
        Finished
    }
``` 

**Impact:** Maybe the `GameState.Revealed` state was intended to be used to indicate that both players have revealed their moves. However, it is not currently being used in the contract, which could lead to confusion or errors in the game logic.

**Recommended Mitigation:** Either remove the `GameState.Revealed` state from the enum or implement it in the game logic to indicate that both players have revealed their moves. This could be done by checking if both players have revealed their moves and then setting the game state to `GameState.Revealed`.


### [L-2] Unspecific and old Solidity Pragma in `RockPaperScissors` and `WinningToken` contracts

**Description:** Consider using a specific version of Solidity in your contracts instead of a wide version. For example, instead of `pragma solidity ^0.8.13;`, use `pragma solidity 0.8.20;` as in the `foundry.toml` file for `solc = "0.8.20"`.

**Impact:** Old solc compiler version could lead to security vulnerabilities and bugs in the contract. Using a specific and recent version of Solidity can help ensure that the contract behaves as expected and is not affected by changes in future versions of the compiler. 

**Recommended Mitigation:** Consider following the `foundry.toml` file for the Solidity version. This will help ensure that the contract is compiled with the same version of Solidity that was used during development and testing.


### [L-3] Unchecked Return in `RockPaperScissors` contract from `winningToken.transferFrom()`

**Description:** Functions `createGameWithToken()` and `joinGameWithToken()` returns a value but it is ignored. Consider checking the return value to ensure that the transfer was successful.
```javascript
    winningToken.transferFrom(msg.sender, address(this), 1);
```

**Impact:** This could lead to a situation where the transfer fails but the function does not handle it, potentially leading to unexpected behavior or loss of funds.

**Recommended Mitigation:** Add a check to ensure that the transfer was successful. This could be done by checking the return value of the `transferFrom()` function and reverting the transaction if it fails.


## Medium

### [M-1] Multiple reentrancy point in `RockPaperScissors` contract can lead to Denial of Service (DoS) attack

**Description:** In the `RockPaperScissors` contract, there are multiple reentrancy points that could be exploited by an attacker, if wrapping the `WinningToken` in a malicious contract was possible. Therefore, the contract is vulnerable to reentrancy attacks, due to these functions that call external contracts and modify state after the call:
- `createGameWithToken()`
- `joinGameWithToken()`
```javascript
    require(
        winningToken.balanceOf(msg.sender) >= 1,
        "Must have winning token"
    );
    // ...
    winningToken.transferFrom(msg.sender, address(this), 1);

    // Modifying state after interactions
    Game storage game = games[gameId];
    game.playerA = msg.sender;
    game.bet = 0; // Zero ether bet because using token
    game.timeoutInterval = _timeoutInterval;
    game.creationTime = block.timestamp;
    game.joinDeadline = block.timestamp + joinTimeout;
    game.totalTurns = _totalTurns;
    game.currentTurn = 1;
    game.state = GameState.Created;

    emit GameCreated(gameId, msg.sender, 0, _totalTurns);

    return gameId;
```

**Impact:** If an attacker can wrap the `WinningToken` in a malicious contract, they could exploit the reentrancy vulnerability to drain funds or manipulate the game state.

**Proof of Concept:**
1. An attacker creates a malicious contract that wraps the `WinningToken` and implements a reentrancy attack on `balanceOf()` and `transferFrom()`.
2. The attacker calls `createGameWithToken()` or `joinGameWithToken()` with the malicious contract as the `winningToken`.
3. The attacker can then exploit the reentrancy vulnerabilities to block the process with reverting calls.
4. No other player can create or join a game until the attacker decides to stop the attack.

**Recommended Mitigation:** Implement the checks-effects-interactions pattern to avoid changing state after external calls.


### [M-2] Fees computation in `RockPaperScissors` when a game is tied can leave ETH dust

**Description:** Calculation of fees in the `RockPaperScissors` contract leaves dust when `_handleTie()` is called. In fact, 10% is taken from the pot for the admin and the rest is split between the players. This is because the division of the pot by 10% for the admin and potentially splitting an odd prize do not take into account the decimal precision of ETH.

```javascript
    // handleTie
    uint256 totalPot = game.bet * 2;
    uint256 fee = (totalPot * PROTOCOL_FEE_PERCENT) / 100;
    uint256 refundPerPlayer = (totalPot - fee) / 2;
```

**Impact:** This can lead to a situation where the contract has a small amount of ETH left over after the fees are taken. This dust can accumulate over time and lead to a significant amount of ETH being left.

**Proof of Concept:**
<details>
<summary>Code</summary>

```javascript
    // Test handling a tie game
    function testAuditTieGameFeeCalculation(uint256 betAmount) public {
        // Limit bet amount to 0.01 - 0.05 ETH (to comply with realistic bet amounts)
        betAmount = bound(betAmount, 0.01 ether, 0.05 ether);
        // Change to 1 turn to make a tie easier to create
        vm.prank(playerA);
        gameId = game.createGameWithEth{value: betAmount}(1, TIMEOUT);

        vm.prank(playerB);
        game.joinGameWithEth{value: betAmount}(gameId);

        // Both players play Rock (creates a tie)
        uint256 playerABalanceBefore = playerA.balance;
        uint256 playerBBalanceBefore = playerB.balance;

        playTurn(
            gameId,
            RockPaperScissors.Move.Rock,
            RockPaperScissors.Move.Rock
        );

        // Verify game state
        (
            ,
            ,
            ,
            ,
            ,
            ,
            ,
            ,
            ,
            ,
            ,
            ,
            ,
            uint8 scoreA,
            uint8 scoreB,
            RockPaperScissors.GameState state
        ) = game.games(gameId);

        assertEq(scoreA, 0);
        assertEq(scoreB, 0);
        assertEq(uint256(state), uint256(RockPaperScissors.GameState.Finished));

        // Verify both players received half of pot minus fees
        uint256 totalPot = betAmount * 2;
        uint256 fee = (totalPot * 10) / 100;
        uint256 refundPerPlayer = (totalPot - fee) / 2;

        assertEq(playerA.balance - playerABalanceBefore, refundPerPlayer);
        assertEq(playerB.balance - playerBBalanceBefore, refundPerPlayer);
        assertEq((refundPerPlayer * 2) + fee, totalPot); // Check total pot
    }
```
</details>

1. PlayerA create a game with a `bet = 0.043784470443721407 ether` and playerB joins the game with the same bet: `pot = 0.087568940887442814 ether`.
2. Both players play the game and the game ends with a tie.
3. The contract takes 10% of the pot for the admin: `fees = 0.008756894088744281 ether`.
4. The rest of the pot is split between the players: `refundPerPlayer = 0.039406023399349266 ether`.
5. The contract returns `0.039406023399349266 ether` to each player, so the total pot minus the fees is `0.078812046798698532 ether`.
6. `0.078812046798698532 ether + 0.008756894088744281 ether = 0.087568940887442813 ether`, which is not equal to the original pot of `0.087568940887442814 ether`.

The difference is `0.000000000000000001 ether`, which is the dust.


**Recommended Mitigation:** Take into account the decimal precision of ETH when calculating fees and splitting the pot.
```javascript
    uint256 fee = ((totalPot * PROTOCOL_FEE_PERCENT * 1e18) / 100) / 1e18;
```
Consider making a common internal function to handle the fees calculation and splitting the pot. This way, you can ensure that the same logic is used in all cases and avoid any discrepancies. Could be useful for `_finishGame()` and maybe fixing the unit tests would be great:
```javascript
    // uint256 expectedPrize = ((BET_AMOUNT * 2) * 90) / 100; // 10% fee // Wrong way
    uint256 totalPot = BET_AMOUNT * 2;
    uint256 fees = ((totalPot * 10) / 100); // 10% fee
    uint256 expectedPrize = totalPot - fees; // 90% of the pot minus fees
```

## High

### [H-1] No limit on game creation in `RockPaperScissors` contract can lead to permanent storage bloat

**Description:** Games are created without any limit. Which means, one user could create an unlimited number of games. With winning tokens, he could create a game for each token, for free. The struct `Game`, which takes up 11 slots, is stored in the contract storage without any cleanup or deletion mechanism. So, if we keep adding entries to the `games` mapping and never remove old/finished ones, your contract's state will keep growing forever.

**Impact:** In a long term, this could lead to bloat in the contract storage, which could make it impossible to interact with. There is a risk of Denial of Service (DoS) if the contract storage becomes too large. This could lead to a situation where users are unable to create or join games, effectively locking them out of the contract.

**Proof of Concept:**

<details>
<summary>Code</summary>

```javascript
    /**
     * @dev This test checks the storage bloat of the game contract when creating multiple games.
     * It creates 100 games and then plays a game to see if the storage layout changes.
     * It also prints the storage layout before and after creating the games.
     * This is useful for auditing purposes to ensure that the contract is not using excessive storage.
     * https://solodit.cyfrin.io/issues/m-02-an-attacker-can-bloat-the-pink-runtime-storage-with-zero-costs-code4rena-phala-network-phala-network-git
     */
    function testAuditCreateMultipleGameStorageBloat() public {
        // Arrange
        vm.prank(address(game));
        token.mint(playerA, 100);
        vm.stopPrank();

        // Log storage of first few games BEFORE creation
        console2.log("### Storage BEFORE creating games ###");
        _printGameMappingStorageLayout();

        // Act
        // If playerA creates 100 games
        vm.startPrank(playerA);
        for (uint256 i = 0; i < 100; i++) {
            token.approve(address(game), 1);
            gameId = game.createGameWithToken(TOTAL_TURNS, TIMEOUT);
        }

        // Log storage of first few games AFTER creation
        console2.log("### Storage AFTER creating games ###");
        _printGameMappingStorageLayout();

        // Continue gameplay as you originally had
        vm.startPrank(playerB);
        token.approve(address(game), 1);
        vm.expectEmit(true, true, false, true);
        emit PlayerJoined(gameId, playerB);
        game.joinGameWithToken(gameId);
        vm.stopPrank();

        (, , , , , , , uint256 totalTurns, , , , , , , , ) = game.games(gameId);

        for (uint256 i = 0; i < totalTurns; i++) {
            playTurn(
                gameId,
                RockPaperScissors.Move.Scissors,
                RockPaperScissors.Move.Paper
            );

            console2.log("### Storage AFTER game turn", i + 1, "###");
            _printGameMappingStorageLayout();
        }

        console2.log("### Storage AFTER finishing a game ###");
        _printGameMappingStorageLayout();

        assertTrue(true);
    }

    /**
     * @dev Prints the storage layout of the game mapping.
     * This is for debugging purposes and should not be used in production.
     */
    function _printGameMappingStorageLayout() internal view {
        console2.log("Game counter: ", game.gameCounter());
        bytes32 base = keccak256(abi.encode(gameId, uint256(0)));
        console2.log("Game ID", uint256(gameId));
        for (uint256 j = 0; j < 12; j++) {
            bytes32 slot = bytes32(uint256(base) + j);
            bytes32 val = vm.load(address(game), slot);

            if (j == 0 || j == 1) {
                address addr = address(uint160(uint256(val)));
                console2.log("Slot", j, "=", addr);
            } else if (j == 9 || j == 10) {
                console2.log("Slot", j, "=");
                console2.logBytes32(val);
            } else if (j == 11) {
                // Solidity packs uint8, enum, and similar small types together into a single slot when possible, to save space.
                // That’s 5 bytes total, and Solidity packs all of that into a single slot (Slot 11).
                console2.log("Slot", j, "=");
                uint8 moveA = uint8(uint256(val) >> (8 * 0));
                uint8 moveB = uint8(uint256(val) >> (8 * 1));
                uint8 scoreA = uint8(uint256(val) >> (8 * 2));
                uint8 scoreB = uint8(uint256(val) >> (8 * 3));
                uint8 state = uint8(uint256(val) >> (8 * 4));

                console2.log("moveA", moveA);
                console2.log("moveB", moveB);
                console2.log("scoreA", scoreA);
                console2.log("scoreB", scoreB);
                console2.log("state", state);
            } else {
                console2.log("Slot", j, "=", uint256(val));
            }
        }
    }
```
</details>


This is how the storage of the `RockPaperScissors` contract looks like:
```bash
  ### Storage BEFORE creating games ###
  Game counter:  0
  Game ID 0
  Slot 0 = 0x0000000000000000000000000000000000000000
  Slot 1 = 0x0000000000000000000000000000000000000000
  Slot 2 = 0
  Slot 3 = 0
  Slot 4 = 0
  Slot 5 = 0
  Slot 6 = 0
  Slot 7 = 0
  Slot 8 = 0
  Slot 9 =
  0x0000000000000000000000000000000000000000000000000000000000000000
  Slot 10 =
  0x0000000000000000000000000000000000000000000000000000000000000000
  Slot 11 =
  moveA 0
  moveB 0
  scoreA 0
  scoreB 0
  state 0
```

The `Game` struct takes up 11 slots, and the `games` mapping is public. This means that every time a game is created, the contract storage grows by 11 slots. If a user creates an unlimited number of games, the contract storage will grow indefinitely.

```bash
    ### Storage AFTER creating games ###
  Game counter:  100
  Game ID 99
  Slot 0 = 0x23223AC37AC99a1eC831d3B096dFE9ba061571CF
  Slot 1 = 0x0000000000000000000000000000000000000000
  Slot 2 = 0
  Slot 3 = 600
  Slot 4 = 0
  Slot 5 = 1
  Slot 6 = 86401
  Slot 7 = 3
  Slot 8 = 1
  Slot 9 =
  0x0000000000000000000000000000000000000000000000000000000000000000
  Slot 10 =
  0x0000000000000000000000000000000000000000000000000000000000000000
  Slot 11 =
  moveA 0
  moveB 0
  scoreA 0
  scoreB 0
  state 0
```

After creating 100 games, the contract storage has grown by 1100 slots. And at the end of each game (or cancelled), the game is not removed from the storage.

```bash
  ### Storage AFTER finishing a game ###
  Game counter:  100
  Game ID 99
  Slot 0 = 0x23223AC37AC99a1eC831d3B096dFE9ba061571CF
  Slot 1 = 0x3d3D63BabfeD85B3e08dE2d4A6c25b0d80cf77f1
  Slot 2 = 0
  Slot 3 = 600
  Slot 4 = 601
  Slot 5 = 1
  Slot 6 = 86401
  Slot 7 = 3
  Slot 8 = 3
  Slot 9 =
  0x8fe307739b80c82d28eabac96094c91e2a3ec596189134a788848aad38618915
  Slot 10 =
  0x97c8d8f22b617f32f21d4f643a756649d13ff7fa3bf0166f78f9f8cc768540bc
  Slot 11 =
  moveA 3
  moveB 2
  scoreA 3
  scoreB 0
  state 3
```

Imagine if a user creates 1000 games, the contract storage will grow by 11000 slots. And if the user never removes the games, the contract storage will keep growing indefinitely.


**Recommended Mitigation:** Consider implementing a cleanup mechanism to remove old or finished games from the contract storage. This could be done by adding an admin function that allows the owner to remove games that are no longer needed. Additionally, consider adding a limit on the number of games that can be created by a single user. You could also track the active entries in the `games` mapping and archive them when they are no longer needed (maybe off-chain).