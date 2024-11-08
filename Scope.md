

Questions
1) how and when is rarity decided?

Scoping
1) ```EnterRaffle``` doesn't check duration so users might be added at any time.
2) ```Refund``` issues? Ln:104 when making address(0) issue and leaving a blank space.
3) ```Refund```: Ln: 102 entrance fee changes after each new user so user could get more than what he paid for
4) SelectWinner function ```deleting```
5) 