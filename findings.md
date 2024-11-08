### [S-#] Looping through players array to check for duplicates in `PuppyRaffle::enterRaffle` is a potential denial of service (DoS) attack. Incrementing gas cost for future entrants

IMPACT: MEDIUM/HIGH

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

Place the followinng test into `PuppyRaffleTest.t.sol`

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