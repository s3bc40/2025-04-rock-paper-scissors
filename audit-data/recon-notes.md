# Recon

## RockPaperScissors

### Notes

- `GameState.Revealed` not used → intentional or miss?
- `uint256 joinDeadline;` related to state var `joinTimeout` which default to `24 hours` , but modifiable by **ADMIN** `RockPaperScissors::setJoinTimeout()`
- `uint256 totalTurns;` should always be odd, even if there is a security with tie internal function
- `Move moveA;` & `Move moveB;` are revealed move from players and stored in game instance. There might be a way to **reveal them after commitment**?
- `joinTimeout` can be modified by admin and admin can **create/join a game**
- `_handleTie` function in case of tie, but odd turns should not happen → try to create a case where it can happen
    - Fees calculation with division by 2 → what happens if the **bet is not even? (decimals)**
    
    ```jsx
    // 1ETH = 1e18 = 1_000_000_000_000_000_000
    // totalPot = 2 * 1e18 = 2e18
    // fee = 10% of 2ETH = 0.2ETH = 200_000_000_000_000_000 => 2e17
    // refundPerPlayer = 900_000_000_000_000_000 => 0.9ETH
    ```
    

### Questions

- *bet is in ETH but we can join with token?*
    - Yes we can just set 0 for game struct: `Game.bet = 0`
- `Game.timeoutInterval` different from `Game.revealDeadline` ?
    - They are related, `Game.revealDeadline` is set after both player have committed a move from `RockPaperScissors::commitMove()`
    
    ```jsx
    game.revealDeadline = block.timestamp + game.timeoutInterval;
    ```
    
- `*GameState.Created`? game should not be in commit phase?* `RockPaperScissors::commitMove()`
    - Normal if it is the first turn and there is a security to wait for the second player
    
    ```jsx
    if (
          game.currentTurn == 1 &&
          game.commitA == bytes32(0) &&
          game.commitB == bytes32(0)
      ) {
          // First turn, first commits
          require(game.playerB != address(0), "Waiting for player B to join");
          game.state = GameState.Committed;
      } else {
          // Later turns or second player committing
          require(game.state == GameState.Committed, "Not in commit phase");
          require(
              game.moveA == Move.None && game.moveB == Move.None,
              "Moves already committed for this turn"
          );
      }
    ```
    

### Possible Issues

- `RockPaperScissors::createGameWithToken()` does not follow CEI → possible reentrance point
- `RockPaperScissors::joinGameWithToken()` does not follow CEI → possible reentrance point
- `RockPaperScissors::commitMove()` does not change the `GameState` if both players have committed
    - could be changed to `GameState.Revealed` because `RockPaperScissors::commitMove()` and `RockPaperScissors::revealMove()` can happen at the same stage (`GameState.Committed`)
- `RockPaperScissors::revealMove()` set to happen at stage `GameState.Committed`
    
    ```jsx
    require(
        msg.sender == game.playerA || msg.sender == game.playerB,
        "Not a player in this game"
    );
    require(game.state == GameState.Committed, "Game not in reveal phase");
    require(
        block.timestamp <= game.revealDeadline,
        "Reveal phase timed out"
    );
    require(_move >= 1 && _move <= 3, "Invalid move");
    ```
    
    - same for `RockPaperScissors::timeoutReveal()` , `RockPaperScissors::canTimeoutReveal()`
- `RockPaperScissors::_finishGame()` internal function has reentrance vulnerability when bet in ETH
    - used in `RockPaperScissors::_determineWinner()` , `RockPaperScissors::timeoutReveal()`

### AI suggestions

- What if `Admin` creates a game, waits till PlayerB commits, then uses `setJoinTimeout()` to grief?
- Can a malicious player commit a hash and never reveal to block game progression?
- Does `timeoutReveal()` fairly refund/reward based on who committed?
- What happens if the bet amount is super small (edge-case rounding issues with fees)?


# AI Audit Notes gas report tests

## 🧨 Potential DoS Attack Surfaces

### 1. Block Gas Limit + Large Game Struct
You’re storing a large Game struct in a public mapping. That means any read/write to it involves a lot of storage ops. This becomes dangerous if:

You loop over games.

You allow untrusted users to create unbounded numbers of games.

➡ Attack vector:
An attacker could spam createGameWithEth or createGameWithToken (which are costly, ~188k–220k gas), inflating the storage and making iteration (if present anywhere, like listing open games or admin cleanups) hit block gas limits or revert.

🛡️ Mitigation:

Rate-limit game creation per address.

Add admin functions to prune stale games.

Avoid public iteration over games.

### 2. DoS via Revert on transferFrom
In joinGameWithToken, you likely have:

solidity
Copy
Edit
winningToken.transferFrom(msg.sender, address(this), 1);
If a malicious token contract is passed, or if a user revokes approval after creating a game, this line will revert.

➡ Attack vector:
A user could create/join a game and never allow the transfer to succeed — the game remains stuck, and others can't interact with it.

🛡️ Mitigation:

Make sure WinningToken is immutable and verified.

Add timeout-based self-recovery for stuck games.

### 3. Reentrancy via Wrapped Token (Theoretical)
Even though WinningToken is clean, an attacker could wrap it in a proxy or malicious contract that performs reentrancy during transferFrom.

➡ Attack vector: If your game logic modifies state after a transferFrom, and that call reenters back into your contract (e.g., calling joinGameWithToken recursively), it could bypass checks or corrupt state.

solidity
Copy
Edit
winningToken.transferFrom(msg.sender, address(this), 1);
game.playerB = msg.sender;
This sequence is vulnerable — write state after external calls is a red flag.

🛡️ Mitigation:

Always update internal state before calling external contracts.

Use nonReentrant modifier.

### 4. Gas Griefing via revealMove
Your revealMove has the highest max gas usage: ~100k+, suggesting dynamic operations. This opens up griefing:

➡ Attack vector:
If a player can craft a situation where revealing becomes expensive, they can cause others to hit gas limits when trying to finalize the game.

🛡️ Mitigation:

Carefully review revealMove and check for O(n) patterns.

Avoid letting one player influence the cost of another’s operations.

## 5. Immutable WinningToken? Still Wrappable
Even if WinningToken is immutable, you can’t stop someone from wrapping it in a malicious contract and using that for transferFrom.

🛡️ Mitigation:

Accept only a known token address.

Add require(msg.sender == tx.origin) on sensitive paths (optional and controversial).

Prefer pull over push design (don’t automatically transfer tokens or ETH to winners).

🚨 Additional Observations
String-based require() messages cost more gas. Not exploitable but worth noting in constrained loops or mass operations.

Consider testing with differential fuzzing — try altering Game parameters massively and see if anything breaks.

https://solodit.cyfrin.io/?b=false&f=&fc=gte&ff=&fn=1&i=HIGH%2CMEDIUM%2CLOW&p=1&pc=&r=all&s=fees+precision&t=