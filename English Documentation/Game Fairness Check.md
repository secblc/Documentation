# Game Fairness Check

The easies way to check the game fairness is to request hash list from Blind Croupier server. Please note that all the information is requested from Ethereum blockchain.

Let us create a link: 

`http://blind-croupier.herokuapp.com/api/v1/transactions?address=**ADDRESS**&gameId=**GAMEID**`.

You need to put your Ethereum wallet address instead of `**ADDRESS**` and the actual id of the game you played instead of `**GAMEID**`.

After you open the link, the server responds with the transaction list of the game: bet, seeds, initial hand, replacement order and the final hand:

```json
{
    "BetSubmitted": {
        "txHash": "0x31a7368260c0a2f8ddad9afecea9ebac1bc97a4274c695fe3ee03b3be9be93a2",
        "bet": 1,
        "level": 1,
        "betPayment": "0.1 eth (1 chips)"
    },
    "CroupierSeedSubmitted: 0": {
        "txHash": "0x0297d8b91e3cea22a3e862e5a146949840e40298c5971cd116fa104bc53352bd",
        "seed": "62504461"
    },
    "CroupierSeedSubmitted: 1": {
        "txHash": "0x4ebae8c3581ed06d6e1e9aa483a363070a9a0c3dd84d77d5ce20c3954f63fc17",
        "seed": "1600494777"
    },
    "PlayerSeedSubmitted: 0": {
        "txHash": "0xe9c4d33b97d3cbaf8ce60a60bbf9f543badfeb43bf2626dd1bf038dcb99ebafd",
        "seed": "393532593"
    },
    "PlayerSeedSubmitted: 1": {
        "txHash": "0x551a2fb28fca743035ffd3eacdec9bb0917b24b74baa2462e35425aef0b7b028",
        "seed": "1301748256"
    },
    "InitialHand": "5♢, 7♢, 9♧, 10♧, 7♡",
    "ReplacementOrderSubmitted": {
        "txHash": "0xa72ded162330c6d60b5de4728e7d216f1b6ae609a04b3854e6d6b9c45bd54e11",
        "replacementOrder": [
            "1",
            "0",
            "1",
            "1",
            "0"
        ]
    },
    "GameFinishedEvent": {
        "txHash": "0x97de8fd8a8af1bd67668cdf49cd89adf74402d1f095fa24a4b9b374b4edf40f2",
        "winCoef": 3,
        "winPayment": "0.3 eth (3 chips)"
    },
    "FinalHand": "3♡, 7♢, J♧, 7♤, 7♡"
}
```

As you can see, it's really easy to retrieve the game from blockchain and check its fairness. We are working on the improved fairness check UI.

# Deep Transaction Study

Deep study requires some advanced tech skills. We are not going to describe basic cryptography or details on some calculations here. You can find extensive explanation in the technical White Paper or ask in the Blind Croupier Slack or Telegram. 

In this guide we are going to briefly describe transactions. To lookup up the method called and its parameters we will use the Etherscan link:

`http://kovan.etherscan.io/tx/**TXHASH**`

where `**TXHASH**` is the transaction hash (unique identifier) contained in the server answer (see above, methods are  (`submitBet`, `submitCrouiperSeedAsSignature`, `submitReplacementOrder` и т.д. )).

After opening the link we are mostly interested in the `Input Data` field, which contains function name and arguments list. We will study all the methods use the same algorithm.

## Step One: Bet Made

To verify bet information, we will need to use the corresponding `txHash` for the `submitBet` method (or `BetSubmitted` event).

`Input Data` contains function parameters:

```
[0]:000000000000000000000000f0005d5b6bf7f42a1e92d0cbbbc75f9773d4158d
[1]:0000000000000000000000001d6acdeb2eada5feca9fe4c78870d0c06f5f1745
[2]:0000000000000000000000000000000000000000000000000000000000000001
[3]:0000000000000000000000000000000000000000000000000000000000000001
[4]:000000000000000000000000000000000000000000000000000000000000001c
[5]:e6c459c79f9ee1c2f1c69858e92cdef31b0287d036b4b37d48059c4410882551
[6]:6b0d4e80a245a78dc2c044a6ad9d6a38c16e338910c621490784e83df74e5334
```

Parameter `[0]` is the Croupier's address:

```
[0]:000000000000000000000000f0005d5b6bf7f42a1e92d0cbbbc75f9773d4158d == 
                              0x5d5b6bf7f42a1e92d0cbbbc75f9773d4158d
```

Parameter `[1]` is the Player's address:

```
0000000000000000000000001d6acdeb2eada5feca9fe4c78870d0c06f5f1745 == 
                      0x1d6acdeb2eada5feca9fe4c78870d0c06f5f1745
```

Parameter `[2]` is the bet made:

```
0000000000000000000000000000000000000000000000000000000000000001 === 1
```

And parameter `[3]` is the level:

```
0000000000000000000000000000000000000000000000000000000000000001 === 1
```

## Step Two: Random Seeds

Like we did in the Step One, use the corresponding `txHash` for the method `submitCrouiperSeedAsSignature` (or `CroupierSeedSubmitted` event).

`https://kovan.etherscan.io/tx/0x0297d8b91e3cea22a3e862e5a146949840e40298c5971cd116fa104bc53352bd`

``InputData``:

```
[0]:000000000000000000000000f0005d5b6bf7f42a1e92d0cbbbc75f9773d4158d
[1]:0000000000000000000000001d6acdeb2eada5feca9fe4c78870d0c06f5f1745
[2]:000000000000000000000000000000000000000000000000000000000000001b
[3]:007169e25b14d495a0a998522d8b02f1aa0121cfd3efa48fe04244d603b9be0d
[4]:6059b49f79bb31333ae34d9456d4e9d16a47bfc44ce803d49ebdf82deb2f64a8
[5]:0000000000000000000000000000000000000000000000000000000000000000
```

We are interested in parameter `[3]` (`r` signature component): 

```
r = 007169e25b14d495a0a998522d8b02f1aa0121cfd3efa48fe04244d603b9be0d
```

To calculate the random seed we need to take last **4 bytes** of this number (i.e. **last 8 hex digits**):

```
seed = 007169e25b14d495a0a998522d8b02f1aa0121cfd3efa48fe04244d603b9be0d ^ ffffff == 03b9be0d
```

Converting the number to the decimal representation (e.g. [here](http://www.binaryhexconverter.com/hex-to-decimal-converter))

```
03b9be0d = 62504461
```

which equals `seed` field in the server provided information

```
"seed": "62504461"
```

Other seeds (Croupier 1, Player 0, Player 1) are calculated the same way.


## Step Three: Replacement Order

Method name: `submitReplacementOrder` (event name: `ReplacementOrderSubmitted`).

Transaction: 
https://kovan.etherscan.io/tx/0xa72ded162330c6d60b5de4728e7d216f1b6ae609a04b3854e6d6b9c45bd54e11


``InputData``:

```
[0]:000000000000000000000000f0005d5b6bf7f42a1e92d0cbbbc75f9773d4158d
[1]:0000000000000000000000001d6acdeb2eada5feca9fe4c78870d0c06f5f1745
[2]:0000000000000000000000000000000000000000000000000000000000000001
[3]:0000000000000000000000000000000000000000000000000000000000000000
[4]:0000000000000000000000000000000000000000000000000000000000000001
[5]:0000000000000000000000000000000000000000000000000000000000000001
[6]:0000000000000000000000000000000000000000000000000000000000000000
[7]:000000000000000000000000000000000000000000000000000000000000001b
[8]:94f391c2a72a80d29c23907a6df4a9505f578acaed8f6c2d1f62e225b6dbcd55
[9]:0781d82e3e5063426d7413285d52941cea761e6da3f649d00391c6af5676a114
```

Parameters `[0]` and `[1]` are the Croupier's and Player's addresses.

Parameters `2` - `6` is the replacement order (5 cards from left to right). `0` means hold, `1` means replace the card.

## Step Five: Submit Order And Game Result

Let's open the last transaction logs  (`GameFinishedEvent`): 

https://kovan.etherscan.io/tx/0x97de8fd8a8af1bd67668cdf49cd89adf74402d1f095fa24a4b9b374b4edf40f2#eventlog

There are two events in the logs:

```
[0] 000000000000000000000000f0005d5b6bf7f42a1e92d0cbbbc75f9773d4158d
[1] 0000000000000000000000001d6acdeb2eada5feca9fe4c78870d0c06f5f1745
[2] 0000000000000000000000000000000000000000000000000429d069189e0000
 		
[0] 0x2e26679f09e74f31e637088f9820b9f39ae1da7796c9ab18fb06227464eeff59
[1] 0x000000000000000000000000f0005d5b6bf7f42a1e92d0cbbbc75f9773d4158d
[2] 0x0000000000000000000000001d6acdeb2eada5feca9fe4c78870d0c06f5f1745
[3] 0x0000000000000000000000000000000000000000000000000000000000000003
 ```

The first one is the MoneyMoved event. Fields from top to bottom:

Croupier's address: 
```
[0] 000000000000000000000000f0005d5b6bf7f42a1e92d0cbbbc75f9773d4158d
```

Player's address: 
```
[1] 0000000000000000000000001d6acdeb2eada5feca9fe4c78870d0c06f5f1745
```

Wei paid (using the same [hex to decimal converter](http://www.binaryhexconverter.com/hex-to-decimal-converter)):

```
[2] 0000000000000000000000000000000000000000000000000429d069189e0000 = 
300000000000000000 Wei = 
0.3 ETH
```

## Checking The Hand Was Generated Properly 

Can we check Croupier and the Bank generated the Hand right?

We have 4 random seeds:

```
62504461 (Croupier 0)
393532593 (Player 0)
1600494777 (Croupier 1)
1301748256 (Player 1)
```

First, let's mix seeds in pairs (Player 0 + Croupier 0, Player 1 + Crouiper 1, using XOR, (^). We can use for example: http://xor.pw/ (use Base 10 for input **and** for ouput). 

The result: 

```
15316196254
83182558959
```

The first seed is used for initial deck shuffling. After that 5 cards are drawn. The rest of the deck is shuffled again using the second seed and the amount of cards required for the replacement is drawn.

To check the final Player's hand given two seeds and the replacement order, we need to call `generateHand` function from the Bank, using seeds and the replacement order as the parameters:

```
generateHand([seed0, seed1], replacementOrder)
```

The result is published in the last line of the initial server reply:

```
"FinalHand": "3♡, 7♢, J♧, 7♤, 7♡"
```

We are working on the tool which can independently do the calculations and prove that everything is calculated correctly.

#### Thank you for attention!
<img align="left" src="https://user-images.githubusercontent.com/31250469/29752217-87d61d60-8b8c-11e7-92fc-5ddf220c5c2b.jpg" width="1200">

#### Best regards,
#### Blind Croupier team
