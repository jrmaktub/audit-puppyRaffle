### [H] Reentranncy attack in `PuppyRaffle:: refund` allows entrant to drain raffle balance

**Description:** The `PuppyRaffle:: refund` function does not follow CEI, and as aresult, enables participants to drain the contract balance.

In the `PuppyRaffle:: refund` function, we first make an external call to the `msg.sender` address and only after making that external call do we update the `PuppyRaffle:: players` array

```javascript
    function refund(uint256 playerIndex) public {
        // @audit MEV
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");
        
        // @audit Reentrancy - making external call before we change state
@>        payable(msg.sender).sendValue(entranceFee);

@>        players[playerIndex] = address(0);
        // @audit low 
        emit RaffleRefunded(playerAddress);
    }
```

A player who was entered the raffle could have a `fallback`/`receive` function that calls the `PuppyRaffle::refund` function again and claim another refund. They could continue the cycle
till the contract balane is drained

**Impact:** All fees paid by the raffle entrants could be stolen

**Proof of Concept:**

1. User enters the raffle.
2. Attacker sets up a contract with a `fallback` function that calls `PuppyRaffle:refund` .
3. Attacker enters the raffle.
4. Attacker calls `PuppyRaffle:refund` from their attack contract, draining the contract.

**Proof of Code:**

<details>

<summary>Code</summary>

Place the following into

```javascript
        function test_reentrancyRefund() public  {

        address[] memory players = new address[](4);
        players[0] = playerOne;
        players[1] = playerTwo;
        players[2] = playerThree;
        players[3] = playerFour;
        puppyRaffle.enterRaffle{value: entranceFee * 4}(players);

        ReentrancyAttacker attackerContract = new ReentrancyAttacker(puppyRaffle);
        address attackUser = makeAddr("attackUser");
        //give attacker money
        vm.deal(attackUser, 1 ether);

        uint256 startingAttackContractBalance = address(attackerContract).balance;
        uint256 startingContractBalance = address(puppyRaffle).balance;

        //attackerContract
        vm.prank(attackUser);
        attackerContract.attack{value: entranceFee }();

        console.log("starting attacker contract balance: ", startingAttackContractBalance);
        console.log("starting contract balance: ", startingContractBalance);

        console.log("ending attacker contract balance: ", address(attackerContract).balance);
        console.log("ending contract balance: ", address(puppyRaffle).balance);

    }
```

</details>

**Recommended Mitigation:**  To prevent this, we should have the ```PuppyRaffle::refund``` function update the `players` array before making the external call. Additionally, we should move the event emission up as well.


### [M-#] Looping through players array to check for duplicates in `PuppyRaffle::enterRaffle` is a potential denial of service (DoS) attack. Incrementing gas cost for future entrants

**Description:** The `PuppyRaffle::enterRaffle` function loops through the `players` array to check for duplicates. However, the longer the `PuppyRaffle::enterRaffle` array is. the more checks a new player will have to make. This means the gas cost for players who enter right when the raffle starts will be automatically lower than those who enter later. Every additional address in the players array is an additional check the loop will have to make.

```javascript
//@audit DoS Attack
        for (uint256 i = 0; i < players.length - 1; i++) {
            for (uint256 j = i + 1; j < players.length; j++) {
                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }
        }

```


**Impact:** The gas cost for entrants will greatly increase as more players enter the raffle. Discouraging later users from entering, and causing a rush at the start of a raffle to be one of the first entrants in the queue.

An attacker might make the `PuppyRaffle::enterRaffle` array so big, no one else ca enter. guaranteeing themselves the win.

**Proof of Concept:**

If we have sets of 100 players enter, the gas costs will be:
- 1st 100 players: 6252128
- 2nd 100 players: 18068218

This is more than 3x more expensive

<details>

<summmary>PoC</summary>

Place the followinng test into `PuppyRaffleTest.t.sol` :

```javascript
    function test_denialOfService() public {
        // address[] memory players = new address[](1);
        // players[0] = playerOne;
        // puppyRaffle.enterRaffle{value: entranceFee}(players);
        // assertEq(puppyRaffle.players(0), playerOne);
        vm.txGasPrice(1);

        //Let's enter 100 players
        uint256 playersNum = 100;
        address[] memory players = new address[](playersNum);
        for (uint256 i=0; i<playersNum; i++){
            players[i] = address(i);
        }

        //see how much gas it costs
        uint256 gasStart = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * players.length }(players);
        uint256 gasEnd = gasleft();

        uint256 gasUsedFirst = (gasStart - gasEnd) * tx.gasprice;

        console.log("Gas cost of the first 100 players", gasUsedFirst);



        //Second 100
        address[] memory playersTwo = new address[](playersNum);
        for (uint256 i=0; i<playersNum; i++){
            playersTwo[i] = address(i + playersNum );
        }

        //see how much gas it costs
        uint256 gasStartSecond = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * players.length }(playersTwo);
        uint256 gasEndSecond = gasleft();

        uint256 gasUsedSecond = (gasStartSecond - gasEndSecond) * tx.gasprice;

        console.log("Gas cost of the second 100 players", gasUsedSecond);

        assert(gasUsedFirst < gasUsedSecond);

    }
```

</details>


**Recommended Mitigation:**

1. Consider allowing duplicates. User can make new wallet address anyways, so a dp
2. Consider using a mapping to check for duplicates. This would allow constant time lookup of whether a user has already entered.

Use OZ Enumerable library.

# Gas

### [G-1] unchanged state variable should be declared constant or immutable

Reading fromm storage is much more expensive than reading from a constant or
immutable variable.

Instances:
- `PuppyRaffle: raffleDuration` should be `immutable`
- `PuppyRaffle:commonImageUri` should be `immutable`


### [G-2] Storage variables in a loop should be cached

```diff
+        uint256 playersLength = players.length
-        for (uint256 i = 0; i < players.length - 1; i++) {
+        for (uint256 i = 0; i < playersLength - 1; i++) {
-            for (uint256 j = i + 1; j < players.length; j++) {
+            for (uint256 j = i + 1; j < playersLength; j++) {
                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }
        }

```


### [I-1]: Solidity pragma should be specific, not wide

Consider using a specific version of Solidity in your contracts instead of a wide version. For example, instead of `pragma solidity ^0.8.0;`, use `pragma solidity 0.8.0;`

<details><summary>1 Found Instances</summary>


- Found in src/PuppyRaffle.sol [Line: 2](src/PuppyRaffle.sol#L2)

	```solidity
	pragma solidity ^0.7.6;
	```

</details>

### [I-1]: Solidity pragma should be specific, not wide


### [I-3:] Missing checks for `address(0)` when assigning values to address state variables

Check for `address(0)` when assigning values to address state variables.

<details><summary>2 Found Instances</summary>


- Found in src/PuppyRaffle.sol [Line: 65](src/PuppyRaffle.sol#L65)

	```solidity
	        feeAddress = _feeAddress;
	```

- Found in src/PuppyRaffle.sol [Line: 209](src/PuppyRaffle.sol#L209)

	```solidity
	        feeAddress = newFeeAddress;
	```

</details>
