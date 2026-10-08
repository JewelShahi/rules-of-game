# Bulgarian Belot (Белот): Complete Rules and Game/UI Logic Specification

> **Purpose:** This document is a developer-facing, deterministic specification for implementing, testing, debugging, or repairing a four-player Bulgarian Belot game and its user interface.
>
> **Language:** The rules are explained in English, while important Bulgarian terms are retained in parentheses.
>
> **Normative words:** **MUST**, **MUST NOT**, **SHOULD**, and **MAY** describe implementation requirements.

---

## 1. Ruleset chosen by this specification

Belot has local, tournament, and website-specific variants in Bulgaria. A game engine cannot safely label behavior simply as “Bulgarian Belot” without fixing the ambiguous options.

This specification uses this **canonical rules profile**:

1. Four players form two fixed teams; partners sit opposite each other.
2. A 32-card deck is used: `7, 8, 9, 10, J, Q, K, A` in each suit.
3. Dealing, bidding, and play proceed **counter-clockwise**. The player to the dealer's right acts first.
4. The bid order is `Clubs < Diamonds < Hearts < Spades < No Trumps < All Trumps`.
5. A player who has passed MAY bid later if the auction is still open.
6. A double (`контра`) is ×2 and a redouble (`реконтра`) is ×4. A later, higher contract cancels the existing double/redouble.
7. At a suit contract, a player void in the led suit MUST trump when an opponent is currently winning, but is free to discard when the player's partner is currently winning.
8. If trump was led, a player following trump MUST overtake the highest trump when possible, even when the partner is winning.
9. At All Trumps, every suit uses trump rank/value and a player following suit MUST raise when possible.
10. At No Trumps, the only card-play obligation is to follow suit.
11. Sequences and four-of-a-kind declarations are disabled at No Trumps.
12. Sequence declarations compete against sequences, and four-of-a-kind declarations compete against four-of-a-kind. A four-of-a-kind does not cancel an opposing sequence merely because it is worth more.
13. Belot is independent of other declarations.
14. The contract-making team must have **strictly more raw points** than the defenders. Equality is a hanging deal (`висяща игра`).
15. The game target is 151 scoreboard points.
16. The match cannot end on capot/valat (`капо`/`валат`), and cannot end while points remain hanging.
17. In this canonical profile, double/redouble multiplies the entire resolved deal, including declarations and capot.

### 1.1 Why these choices must be explicit

Bulgarian sources disagree on some details. For example, some describe counter-clockwise play beginning to the dealer's right, while another online implementation uses clockwise play beginning to the dealer's left. Some tournament rules cancel a double when capot occurs, while other published rules multiply the capot. Some rules compare sequence and four-of-a-kind declarations separately, while others compare all non-belot declarations in one pool.

Therefore, a production game SHOULD store a versioned `rulesProfileId`, not only a language or country label. Section 18 lists recommended compatibility switches.

---

## 2. Core vocabulary

| English term | Bulgarian term | Meaning |
|---|---|---|
| deal / hand | раздаване | One distribution of all 32 cards, its auction, eight tricks, and scoring |
| trick | взятка / ръка | Four cards, one played by each player |
| lead suit | искан цвят | Suit of the first card in a trick |
| trump | коз | Suit that beats every non-trump card in a suit contract |
| to trump / ruff | цакам | Play a trump because one cannot follow the led suit |
| overtrump | надцаквам | Play a higher trump than the highest trump already in the trick |
| undertrump | подцаквам | Play a lower trump than one already in the trick |
| bid / contract call | анонс | Call naming the game type during the auction |
| declaration / meld | обява | Bonus combination such as a tierce, quarte, quinte, or carré |
| declaring team | обявил отбор | Team that made the final/highest contract bid |
| defenders | противници | Other team |
| made / outside | изкарана / вън | Declaring team has more raw points |
| inside / set | вкарана / вътре | Declaring team has fewer raw points |
| hanging | висяща | Raw points are equal |
| double | контра | Opponents challenge the contract; deal multiplier becomes ×2 |
| redouble | реконтра | Declaring team answers the double; multiplier becomes ×4 |
| capot / valat | капо / валат | One team wins all eight tricks |
| last ten | последно 10 | Ten raw points for winning trick eight |

**Naming warning:** Bulgarian players may use `анонс` for an auction bid and `обява` for a card combination. The UI SHOULD use different internal enums even if player-facing labels vary.

---

## 3. Players, teams, seats, and objective

- Exactly **4 players** participate.
- Seats are fixed for the match.
- Opposite players are partners.
- Recommended seat mapping if seat indexes increase clockwise:
  - Team A: seats `0` and `2`
  - Team B: seats `1` and `3`
- Under the canonical counter-clockwise profile:
  - `nextSeat(s) = (s + 3) mod 4`
  - first bidder and first trick leader = `nextSeat(dealer)`
  - next dealer after any attempted deal = `nextSeat(dealer)`
- Normal winning target: **151 or more scoreboard points**, subject to the finish rules in Section 14.

A deal has exactly eight tricks. Each player plays exactly one card per trick and all 32 cards must be consumed in a completed deal.

---

## 4. Deck model

### 4.1 Cards

The deck contains eight ranks in each of four suits:

- Clubs `♣`
- Diamonds `♦`
- Hearts `♥`
- Spades `♠`

Ranks:

`7, 8, 9, 10, J, Q, K, A`

Recommended stable IDs:

```text
C7 C8 C9 CT CJ CQ CK CA
D7 D8 D9 DT DJ DQ DK DA
H7 H8 H9 HT HJ HQ HK HA
S7 S8 S9 ST SJ SQ SK SA
```

Do not encode rank strength into the card ID. Strength changes with the contract.

### 4.2 Trump rank and point value

From strongest to weakest:

| Rank | Trick strength | Raw card points |
|---|---:|---:|
| J | 8 | 20 |
| 9 | 7 | 14 |
| A | 6 | 11 |
| 10 | 5 | 10 |
| K | 4 | 4 |
| Q | 3 | 3 |
| 8 | 2 | 0 |
| 7 | 1 | 0 |

One complete suit evaluated as trump contains **62 raw card points**.

### 4.3 Non-trump rank and point value

From strongest to weakest:

| Rank | Trick strength | Raw card points |
|---|---:|---:|
| A | 8 | 11 |
| 10 | 7 | 10 |
| K | 6 | 4 |
| Q | 5 | 3 |
| J | 4 | 2 |
| 9 | 3 | 0 |
| 8 | 2 | 0 |
| 7 | 1 | 0 |

One complete suit evaluated as non-trump contains **30 raw card points**.

### 4.4 Contract totals before declarations

| Contract | Card total before last trick | Last-trick bonus | Intrinsic transform | Deal base before capot |
|---|---:|---:|---:|---:|
| One suit as trump | 152 | 10 | none | 162 raw ≈ 16 score |
| No Trumps | 120 | 10 | double card/last-trick total | 260 raw = 26 score |
| All Trumps | 248 | 10 | none | 258 raw ≈ 26 score |

For No Trumps, the ×2 shown here is an **intrinsic scoring rule**, not `контра`. Apply it before rounding and before a possible double/redouble multiplier.

---

## 5. Match and deal state machine

A robust engine SHOULD use explicit states:

```text
MATCH_SETUP
  -> DEAL_INITIAL_5
  -> AUCTION
      -> REDEAL_ALL_PASS
      -> DEAL_REMAINING_3
  -> TRICK_PLAY (8 tricks)
  -> DECLARATION_RESOLUTION
  -> DEAL_SCORING
  -> MATCH_END_CHECK
      -> DEAL_INITIAL_5
      -> MATCH_FINISHED
```

Recommended substate fields:

```yaml
rulesProfileId: bg-canonical-v1
targetScore: 151
seats: [player0, player1, player2, player3]
teams:
  A: [0, 2]
  B: [1, 3]
dealerSeat: 0
turnSeat: 3
hands: {0: [], 1: [], 2: [], 3: []}
auction:
  highestContract: null
  highestBidderSeat: null
  multiplier: 1
  passesSinceLastLiveCall: 0
  initialPassCount: 0
contract: null
tricks: []
currentTrick: []
declarations: []
hangingScore: 0
matchScore: {A: 0, B: 0}
finishBlockedByCapot: false
```

**Important:** Store raw points and scoreboard points separately. Never overwrite raw totals with rounded totals before deciding whether the contract is made, inside, or hanging.

---

## 6. Dealing

### 6.1 Start of a match

The first dealer may be selected randomly or by a predefined lobby rule. The choice MUST be visible and deterministic after selection.

### 6.2 First stage: five cards

1. Shuffle the full 32-card deck.
2. Starting with the player to the dealer's right, deal counter-clockwise.
3. Give each player **3 cards**.
4. In the same seat order, give each player **2 more cards**.
5. Every hand now contains 5 cards.
6. Start the auction with the player to the dealer's right.

### 6.3 Second stage: three cards

After the auction ends with a contract:

1. Deal **3 additional cards** to each player in the same direction and starting position.
2. Each player now has exactly 8 cards.
3. No card remains undealt.
4. The first trick is led by the player to the dealer's right, not by the final bidder.

### 6.4 All four players pass

If the first four calls are all `PASS` and no contract was ever made:

- Abort the deal before the remaining 12 cards are dealt.
- Award no points.
- Do not change the hanging pool.
- Advance the dealer.
- Shuffle and deal again.
- An all-pass redeal does not satisfy any “one more deal after capot” requirement.

### 6.5 Deal validation and misdeal handling

A digital game MUST prevent or detect:

- duplicate cards;
- missing cards;
- wrong hand sizes;
- cards owned by multiple players;
- a card being played twice;
- remaining cards being dealt before a valid contract exists;
- play beginning with fewer or more than eight cards per player.

Recommended behavior is transaction rollback and redeal, with an audit message. A digital game SHOULD NOT invent a point penalty unless the selected tournament profile defines one.

---

## 7. Auction (bidding)

### 7.1 Contract order

From lowest to highest:

```text
1 Clubs        ♣
2 Diamonds     ♦
3 Hearts       ♥
4 Spades       ♠
5 No Trumps    NT / Без коз
6 All Trumps   AT / Всичко коз
```

Suits in the same row are **not equal**; the listed suit order is part of Bulgarian bidding.

### 7.2 Normal bid legality

On a player's turn, a normal contract bid is legal only if:

- no contract exists yet; or
- its order is strictly higher than the current contract.

A player cannot repeat or lower the current contract.

A player who passed earlier is not eliminated and MAY make a later higher bid.

### 7.3 Pass

`PASS` is always legal during the auction, except that a UI MAY auto-pass a player who has no other legal action.

Do not confuse:

- four initial passes: cancel and redeal;
- three consecutive passes after a live call: end the auction.

### 7.4 Double (`контра`)

A double is legal only when all are true:

1. A normal contract currently exists.
2. Current multiplier is `1`.
3. The acting player's team is the opponent of the team that made the current highest bid.
4. The auction has not ended.

Effect:

- multiplier becomes `2`;
- current contract and current highest bidder remain unchanged;
- the double becomes the latest live call for purposes of counting subsequent passes.

### 7.5 Redouble (`реконтра`)

A redouble is legal only when all are true:

1. Current multiplier is `2`.
2. The acting player's team is the team that made the current highest bid.
3. The auction has not ended.

Effect:

- multiplier becomes `4`;
- contract and highest bidder remain unchanged;
- the redouble becomes the latest live call.

No multiplier above ×4 exists in this profile.

### 7.6 Higher bid after double or redouble

Any player may make a legal higher normal contract when it is their turn. If that happens:

- replace the contract;
- set the new bidder and declaring team;
- reset multiplier to `1`;
- erase the effect of the old double/redouble;
- allow the new contract to be doubled later.

Example:

```text
Seat 3: Hearts
Seat 2: Double
Seat 1: Spades
```

The final active contract is Spades at ×1. The double of Hearts does not carry over.

### 7.7 Ending the auction

After any live call (normal bid, double, or redouble), the auction ends after the other three seats consecutively pass.

Implementation-safe rule:

```pseudo
if action is BID or DOUBLE or REDOUBLE:
    passesSinceLastLiveCall = 0
else if action is PASS:
    passesSinceLastLiveCall += 1

if highestContract is null and initialPassCount == 4:
    outcome = ALL_PASS
else if highestContract is not null and passesSinceLastLiveCall == 3:
    finalizeAuction()
```

### 7.8 Auction UI requirements

The UI MUST:

- enable only bids above the current bid;
- enable Double only for the correct team and only at ×1;
- enable Redouble only for the declaring team and only at ×2;
- show the current contract, bidder/team, multiplier, and whose turn it is;
- reset the multiplier display if a higher contract replaces the doubled contract;
- preserve a complete action history;
- not expose the final three cards before the auction ends.

---

## 8. Trick play: concepts common to all contracts

1. The first leader is the player to the dealer's right.
2. The leader may play any card in hand.
3. The first card establishes the led suit.
4. Each other player plays one legal card in turn.
5. After four cards, determine the winner.
6. The winner leads the next trick.
7. After eight tricks, score the deal.

A card from a suit other than the led suit cannot win unless it is an actual trump in a one-suit trump contract.

### 8.1 Current trick winner

The engine needs the **current** winner before validating the third and fourth plays, because a player void in the led suit may be required to trump only when an opponent is winning.

### 8.2 Completed trick winner

- Suit contract: highest trump wins if any trump was played; otherwise highest card of the led suit wins.
- No Trumps: highest card of the led suit in non-trump rank wins.
- All Trumps: highest card of the led suit in trump rank wins. Off-suit cards never trump the led suit.

---

## 9. Legal-card rules by contract

## 9.1 One-suit trump contract

Apply the following in order.

### Case A: player has at least one card of the led suit

The player MUST follow suit.

- If the led suit is not trump: any card of the led suit is legal; there is no obligation to beat a non-trump card.
- If the led suit is trump:
  - if the player can beat the highest trump currently in the trick, the player MUST play a higher trump;
  - otherwise, any remaining trump is legal.

The duty to raise when trump is led applies even if the partner currently holds the trick.

### Case B: player is void in the led suit and partner is currently winning

The player MAY play any card:

- discard a non-trump;
- play a lower trump;
- play a higher trump.

There is no duty to trump or overtrump when the partner is currently winning.

### Case C: player is void in the led suit and an opponent is currently winning

- If the player has no trump: any card is legal.
- If no trump has yet been played in the trick: the player MUST play a trump.
- If a trump has already been played by an opponent:
  - if the player has a higher trump, the player MUST overtrump;
  - if the player has no higher trump, the player MAY play any card. This profile does **not** force an undertrump.

### 9.1.1 Important examples

1. Hearts are trump. Clubs are led. You have a club: you MUST play a club, even if you also hold the jack of hearts.
2. Hearts are trump. Clubs are led. You have no club. Your partner is winning with the ace of clubs: any card is legal.
3. Hearts are trump. Clubs are led. You have no club. An opponent is winning and no trump is on the table: you MUST play a heart if you have one.
4. Hearts are trump. Clubs are led. An opponent has already played the nine of hearts. You have the jack of hearts: you MUST play it.
5. Same trick, but your only heart is the seven: because it cannot overtrump the nine, this profile allows either the seven of hearts or any discard.
6. Hearts are led as trump. The nine of hearts is currently highest. You hold jack and seven of hearts: only the jack is legal.
7. Hearts are led as trump. The jack of hearts is currently highest. You hold nine and seven of hearts: either heart is legal because overtrumping is impossible.

## 9.2 All Trumps (`Всичко коз`)

Every suit uses trump rank and trump card values, but no suit globally trumps another suit.

Rules:

1. If the player has the led suit, the player MUST follow it.
2. If the player can beat the highest card of that led suit already played, the player MUST do so.
3. The raise obligation applies even when the partner is winning.
4. If the player has the led suit but cannot raise, any card of that suit is legal.
5. If the player is void in the led suit, any card is legal.
6. An off-suit card cannot win the trick.

Example: Clubs are led. The nine of clubs is highest. A player holding jack of clubs and seven of clubs MUST play jack of clubs. A player with no clubs can discard any suit, but that off-suit card cannot win.

## 9.3 No Trumps (`Без коз`)

1. If the player has the led suit, the player MUST follow it.
2. If the player is void in the led suit, any card is legal.
3. There is no obligation to beat the current highest card.
4. There is no trumping, overtrumping, or cross-suit winning.
5. Card rank is always the non-trump rank.

### 9.4 Reference legal-card algorithm

```pseudo
function legalCards(hand, trick, contract, actorTeam):
    if trick.isEmpty:
        return hand

    ledSuit = trick[0].card.suit
    follow = cards(hand, suit = ledSuit)

    if contract.type == NO_TRUMPS:
        return follow if follow.notEmpty else hand

    if contract.type == ALL_TRUMPS:
        if follow.isEmpty:
            return hand
        highestLed = highestCardOfSuit(trick, ledSuit, TRUMP_ORDER)
        higher = cards(follow, rankHigherThan = highestLed, order = TRUMP_ORDER)
        return higher if higher.notEmpty else follow

    # One-suit trump contract
    trumpSuit = contract.suit

    if follow.notEmpty:
        if ledSuit != trumpSuit:
            return follow
        highestTrump = highestCardOfSuit(trick, trumpSuit, TRUMP_ORDER)
        higher = cards(follow, rankHigherThan = highestTrump, order = TRUMP_ORDER)
        return higher if higher.notEmpty else follow

    currentWinner = determineCurrentWinner(trick, contract)
    if teamOf(currentWinner.seat) == actorTeam:
        return hand

    trumps = cards(hand, suit = trumpSuit)
    if trumps.isEmpty:
        return hand

    trickTrumps = cards(trick, suit = trumpSuit)
    if trickTrumps.isEmpty:
        return trumps

    highestTrump = highest(trickTrumps, TRUMP_ORDER)
    higher = cards(trumps, rankHigherThan = highestTrump, order = TRUMP_ORDER)
    return higher if higher.notEmpty else hand
```

The server MUST re-run this function. Client-side disabled cards are not a security or correctness boundary.

---

## 10. Declarations (`обяви`)

Declarations are bonus combinations in the final eight-card hand. They are different from auction bids.

### 10.1 Timing

- A player declares all tierces, quartes, quintes, and four-of-a-kind combinations when playing that player's **first card of the deal**.
- A missed declaration cannot be added later.
- The declared combination need not contain the first card played.
- The engine SHOULD record the claim atomically with the first-card action.
- In a human-like mode, exact cards may be revealed after all four players have made their first play; in a fully automatic rules mode, the server may calculate and resolve declarations automatically.

### 10.2 Sequence rank order

For declarations only, ranks use their natural consecutive order:

```text
7 < 8 < 9 < 10 < J < Q < K < A
```

Trump trick rank does not affect sequence order. `J-9-A` is not a sequence.

### 10.3 Sequence types

A sequence must be in one suit.

| Name | Cards | Raw bonus |
|---|---:|---:|
| Tierce (`терца`) | exactly/maximal run of 3 | 20 |
| Quarte (`кварта`, often announced “50”) | exactly/maximal run of 4 | 50 |
| Quinte (`квинта`, often announced “100”) | run of 5 or more | 100 |

A run of six, seven, or eight cards is still one quinte worth 100, unless a selected house profile says otherwise.

Use maximal runs. Example: `8-9-10-J-Q` is one quinte, not a tierce plus a quarte plus a quinte.

A hand may contain more than one disjoint sequence, including sequences in different suits.

### 10.4 Four of a kind (`каре`)

| Four of a kind | Raw bonus | Relative set strength |
|---|---:|---:|
| Four Jacks | 200 | highest |
| Four Nines | 150 | second |
| Four Aces | 100 | next |
| Four Tens | 100 | next |
| Four Kings | 100 | next |
| Four Queens | 100 | next |
| Four Eights | invalid | — |
| Four Sevens | invalid | — |

Among the 100-point sets, tie-breaking order is `A > 10 > K > Q`, based on trump point value.

### 10.5 Card overlap

Canonical rule:

- A card cannot simultaneously belong to an ordinary sequence and a four-of-a-kind declaration by the same player.
- If overlap exists, the player must choose a valid declaration plan.
- Belot is independent and MAY overlap with a sequence.
- A single maximal sequence is not split into overlapping smaller sequences.

The UI MUST ask for a choice when multiple legal declaration plans exist, unless an “automatic best declaration plan” option is explicitly enabled.

### 10.6 No-Trumps restriction

At No Trumps:

- tierce, quarte, quinte, and four-of-a-kind declarations are disabled;
- belot is impossible because there is no trump suit;
- last ten and capot still apply.

### 10.7 Resolving competing sequences

Compare only the strongest declared sequence from each team:

1. Longer sequence wins: quinte > quarte > tierce.
2. If lengths are equal, the sequence with the higher top card wins.
3. Suit does not break a tie.
4. If both teams' strongest sequences have the same length and same top rank, all ordinary sequence bonuses for both teams are canceled.
5. Otherwise, the winning team scores **all** its valid sequence declarations, not only the declaration used to win the comparison.
6. The losing team scores none of its sequence declarations.

Examples:

- Team A has `7-8-9-10`; Team B has `Q-K-A`. Team A wins because four cards beat three.
- Both teams have a four-card sequence ending at K. Suit is ignored, so all sequence bonuses are canceled.
- Team A has a quinte plus a tierce; Team B has only a quarte. Team A wins and scores both Team A sequences.

### 10.8 Resolving competing four-of-a-kind declarations

Compare only the strongest set from each team using:

```text
Jacks > Nines > Aces > Tens > Kings > Queens
```

- The team with the strongest set scores all its valid set declarations.
- The other team scores none of its set declarations.
- Set comparison is separate from sequence comparison in this profile.
- Therefore, Team A may win the sequence category while Team B wins the set category.

### 10.9 Belot (`белот`)

Belot is king plus queen of a trump suit held by the same player.

- In a one-suit trump contract, only `K + Q` of the chosen trump suit qualifies.
- In All Trumps, each suit is a trump suit, so each same-suit `K + Q` pair qualifies separately.
- In No Trumps, no belot exists.
- Each belot is worth 20 raw points.
- Belot does not take part in sequence or set comparisons.
- Belot may overlap with a sequence.
- A team can score multiple belots in All Trumps.

Declaration timing:

1. When the first of the pair is legally played, the player must announce `Belot`.
2. The second may be announced as `Rebelot` for UI feedback.
3. Score the pair once, not once per card.
4. If the first card is played without the required declaration in manual mode, the bonus is lost.
5. The engine must never allow declaration text to make an otherwise illegal card play legal.

Belot is “always valid” in the sense that it is not canceled by a stronger opposing sequence or set. It is still part of the team's deal total and therefore follows the final contract settlement: a team that is inside or loses a doubled deal can still receive zero scoreboard points.

### 10.10 Capot and declarations

Canonical handling:

- The team that wins no tricks loses its ordinary sequence and set bonuses.
- Belot remains a separately valid bonus when properly declared, but final inside/double settlement may transfer or erase its scoreboard award.
- This behavior MUST be covered by tests because some tables use different house rules.

---

## 11. Raw point calculation

Use these stages in this exact order.

### 11.1 Captured card points

For every completed trick, add the four card values to the team that won the trick, using the active contract's value table.

### 11.2 Last trick

Add 10 raw points to the team that wins trick 8.

### 11.3 No-Trumps intrinsic doubling

At No Trumps, multiply each team's captured-card-plus-last-ten subtotal by 2.

Do not apply this to capot's extra 90. Ordinary declarations and belot do not exist at No Trumps in this profile.

### 11.4 Add valid declarations

Add resolved sequence, set, and belot bonuses to the relevant team's raw total.

### 11.5 Capot

If a team won all eight tricks:

- add 90 raw points to that team;
- mark `isCapot = true`;
- invalidate the trickless team's ordinary sequence/set bonuses as described above.

Base capot totals without declarations:

| Contract | Calculation | Raw total | Scoreboard total before kontra |
|---|---|---:|---:|
| Suit | 162 + 90 | 252 | 25 |
| No Trumps | 260 + 90 | 350 | 35 |
| All Trumps | 258 + 90 | 348 | 35 |

### 11.6 Contract comparison uses raw totals

Let:

```text
D = declaring team's raw total
F = defending team's raw total
```

Compare before division or rounding:

- `D > F`: contract made (`изкарана`)
- `D < F`: declaring team inside (`вътре`)
- `D == F`: hanging (`висяща`)

Never compare rounded values. Example: 134 versus 124 at All Trumps may display as 13–13 after special rounding, but 134 is still the raw winner.

---

## 12. Rounding to scoreboard points

Bulgarian Belot keeps scores in units of approximately ten raw points.

### 12.1 Suit contract and No Trumps

After the No-Trumps intrinsic doubling, use:

```pseudo
roundSuitOrNoTrump(raw):
    tens = floor(raw / 10)
    units = raw mod 10
    return tens + (1 if units >= 6 else 0)
```

Thus:

- 87 → 9
- 75 → 7
- 145 → 14
- 146 → 15

This “round upward from 6” convention avoids silently applying generic programming-language rounding.

### 12.2 All Trumps

Normally use:

```pseudo
roundAllTrump(raw):
    tens = floor(raw / 10)
    units = raw mod 10
    return tens + (1 if units >= 5 else 0)
```

### 12.3 Special All-Trumps `...4 / ...4` case

Because the base raw total is 258, both teams may end in 4; for example, 134–124. If both were rounded normally, both would round down and the deal would total only 25 instead of 26.

Canonical correction:

- the team with fewer raw points rounds up;
- the team with more raw points rounds down.

Example:

```text
134 -> 13
124 -> 13 (special upward correction for the lower raw total)
```

The contract winner is still the team with 134 raw points.

### 12.4 Hanging tie rounding

When `D == F` and the deal is not doubled:

- defending team's current award = `floor(F / 10)`;
- declaring team's hanging addition = `ceil(D / 10)`;
- if the value is already divisible by 10, floor and ceiling are equal.

Examples:

- 106–106 in a suit contract with declarations: defenders write 10; 11 hang.
- 154–154 at All Trumps with declarations: defenders write 15; 16 hang.
- 130–130 at No Trumps: defenders write 13; 13 hang.

Do not use ordinary symmetric rounding for a tie.

### 12.5 Total deal score for inside/double logic

When one team is to receive the whole deal, calculate a single deal total rather than summing two independently rounded team values:

```pseudo
dealTotalUnits = roundAccordingToContract(D + F)
```

Equivalent practical base values without declarations are:

- normal suit deal: 16
- normal No-Trumps deal: 26
- normal All-Trumps deal: 26
- suit capot: 25
- No-Trumps capot: 35
- All-Trumps capot: 35

Each 10 raw declaration points adds 1 scoreboard point before a double/redouble. Because declaration values are multiples of 10, this is exact.

---

## 13. Settling a deal

Let:

- `M` = 1, 2, or 4 auction multiplier;
- `H` = hanging scoreboard points carried from earlier deals;
- `winner` = team with higher raw total when totals are unequal.

### 13.1 Undoubled, contract made (`M = 1`, `D > F`)

- Declaring team receives its own rounded scoreboard total.
- Defenders receive their own rounded scoreboard total.
- Declaring team also receives all previous hanging points `H` because it won the deal.
- Set `H = 0`.

### 13.2 Undoubled, declaring team inside (`M = 1`, `D < F`)

- Declaring team receives 0 for the current deal.
- Defenders receive the entire rounded deal total, including the declaring team's valid declarations.
- Defenders also receive all previous hanging points `H`.
- Set `H = 0`.

### 13.3 Undoubled, hanging (`M = 1`, `D == F`)

- Defenders receive the floor-rounded defending share now.
- Declaring team's ceiling-rounded share is added to `H`.
- Existing `H` remains and accumulates.
- No team wins the deal for the purpose of collecting previous hanging points.

### 13.4 Doubled or redoubled, unequal totals (`M = 2 or 4`)

The deal is all-or-nothing:

- Team with higher raw total receives `dealTotalUnits × M`.
- Other team receives 0.
- Winner also receives previous hanging points `H`; `H` itself is not multiplied again.
- Set `H = 0`.

This applies whether the winner is the declaring team or the defenders.

### 13.5 Doubled or redoubled, hanging (`M = 2 or 4`, `D == F`)

- Neither team receives current-deal points.
- Add `dealTotalUnits × M` to `H`.
- Preserve any hanging points already in `H`.

### 13.6 Reference settlement pseudocode

```pseudo
function settleDeal(rawA, rawB, declaringTeam, contract, multiplier,
                    existingHanging, isCapot):
    defendingTeam = otherTeam(declaringTeam)
    D = raw[declaringTeam]
    F = raw[defendingTeam]
    award = {A: 0, B: 0}
    H = existingHanging

    if D == F:
        if multiplier == 1:
            award[defendingTeam] += floor(F / 10)
            H += ceil(D / 10)
        else:
            H += totalDealUnits(rawA + rawB, contract, isCapot) * multiplier
        return {award, hanging: H, result: HANGING}

    winner = declaringTeam if D > F else defendingTeam

    if multiplier == 1 and winner == declaringTeam:
        award[declaringTeam] += roundTeam(D, contract, rawA, rawB)
        award[defendingTeam] += roundTeam(F, contract, rawA, rawB)
    else:
        award[winner] += totalDealUnits(rawA + rawB, contract, isCapot) * multiplier

    award[winner] += H
    H = 0

    return {
        award,
        hanging: H,
        result: MADE if winner == declaringTeam else INSIDE,
        winner
    }
```

`totalDealUnits` and `roundTeam` must implement Sections 12.1–12.5, not generic `Math.round`.

---

## 14. End of the match

Check match completion only after the deal has been fully scored.

### 14.1 Normal finish

A team is eligible to win when:

- its cumulative score is at least the target, normally 151;
- the hanging pool is zero;
- the just-completed deal was not capot;
- any prior capot finish lock has been cleared by this resolved non-capot, non-hanging deal.

### 14.2 Both teams reach the target

If both are at or above 151 after an eligible deal:

- higher cumulative score wins;
- if scores are equal, continue playing.

### 14.3 “You cannot go out with capot” (`С капо не се излиза`)

If a capot causes a team to reach or pass 151:

- do not finish the match;
- set `finishBlockedByCapot = true`;
- play at least one later resolved non-capot deal.

An all-pass redeal does not clear the lock. Another capot does not clear it. A hanging deal SHOULD NOT clear it because no final deal winner exists and unresolved points remain.

### 14.4 Hanging points block the finish

A match cannot finish while `hangingScore > 0`, even if one team is already at or above 151. Continue until a later non-tied deal awards the hanging pool.

### 14.5 Suggested end-check algorithm

```pseudo
if deal.isCapot:
    finishBlockedByCapot = true
    continueMatch()
else if deal.result == HANGING:
    continueMatch()
else:
    finishBlockedByCapot = false

    if hangingScore != 0:
        error("non-tied deal must resolve hanging points")

    if scoreA < target and scoreB < target:
        continueMatch()
    else if scoreA == scoreB:
        continueMatch()
    else:
        winner = A if scoreA > scoreB else B
        finishMatch(winner)
```

---

## 15. Worked scoring examples

### Example 1: Suit contract made normally

- Hearts contract, ×1.
- Declaring team raw total: 87.
- Defenders raw total: 75.
- No hanging points.

`87 > 75`, so the contract is made.

Scoreboard result: 9–7.

### Example 2: Suit contract inside

- Declaring team: 75 raw.
- Defenders: 87 raw.

`75 < 87`, so the declaring team is inside. The defenders receive the entire 162-point base deal: 16 scoreboard points. Result: 0–16 from the current deal.

### Example 3: Made contract with a declaration

- Suit contract.
- Declaring team captures 92 raw card/last-trick points and wins a tierce worth 20.
- Defenders capture 70.
- Final raw comparison: 112–70.

Contract is made. Team rounding produces 11 and 7. The deal adds 18 scoreboard points in total because declarations add 2.

### Example 4: Declaration turns apparent card win into inside

- Suit contract.
- Declaring team captures 90 card/last-trick points.
- Defenders capture 72 and own the winning quarte worth 50.
- Final raw totals: 90–122.

The declaring team is inside. Do not compare only captured cards. Defenders receive the whole deal total: `(162 + 50) / 10`, rounded as the contract requires = 21.

### Example 5: No-Trumps scoring

- Before the intrinsic No-Trumps double, captured-card-plus-last-ten totals are 68–62.
- Transform them to 136–124.
- Contract is made.
- Scoreboard result: 14–12.

Do not round 68–62 first and then double; that can produce a different result in other splits.

### Example 6: All-Trumps special 4/4 rounding

- Final raw totals: 134–124.
- Raw winner is the 134 team.
- Display/score 134 as 13.
- Special-case the lower 124 upward to 13.

The displayed result is 13–13, but it is **not** a hanging deal because raw totals are unequal.

### Example 7: Undoubled hanging deal

- Declaring and defending teams both have 106 raw points.
- Defenders write 10 now.
- 11 are added to the hanging pool.
- The declaring team writes 0 now.

A later non-tied deal winner receives the 11 hanging points.

### Example 8: Inside with existing hanging points

- `H = 11` from a previous deal.
- Current suit deal: declaring team 70, defenders 92.
- Defenders win 16 for the current deal and collect 11 hanging.

Current awards: declaring team 0, defenders 27. New hanging pool: 0.

### Example 9: Double

- Suit deal, ×2.
- Declaring team 90 raw; defenders 72 raw.
- Declaring team wins.
- Total base deal is 16 scoreboard points.

Award: declaring team 32, defenders 0.

### Example 10: Redouble with declarations

- All Trumps, ×4.
- Total valid declaration bonuses across the deal: 70 raw points.
- Base deal is 26 scoreboard points; declarations add 7.
- Winner receives `(26 + 7) × 4 = 132`.
- Loser receives 0.

### Example 11: No-Trumps capot

- Captured cards plus last ten produce 260 after No-Trumps intrinsic doubling.
- Add 90 capot bonus, not another intrinsic ×2.
- Base capot = 350 raw = 35 scoreboard points.
- At counter ×2, this profile awards 70 to the winner.
- The match still cannot end on that capot.

### Example 12: Multiple hanging deals

- Existing `H = 11`.
- Next deal hangs at ×2 and its total doubled value is 42.
- New `H = 53`.
- Next non-tied deal is won by Team B.

Team B receives its normal current-deal award plus 53. Only then can the match finish.

---

## 16. Edge-case catalogue

This section is intended as a bug checklist.

### 16.1 Auction edge cases

- First three players pass; dealer may still bid.
- A player passes, another player bids, and the first player later raises: legal.
- A player tries to repeat the current bid: illegal.
- A player tries a lower bid: illegal.
- Declaring team's player tries to double own contract: illegal.
- Defending player tries to redouble: illegal.
- Redouble attempted without a live double: illegal.
- A higher bid after double resets multiplier to ×1.
- A higher bid after redouble resets multiplier to ×1.
- Three passes after a double end the auction at ×2.
- Three passes after a redouble end it at ×4.
- Four initial passes trigger redeal, no score, dealer advances.
- An All-Trumps bid leaves no higher normal bid; only legal team-dependent double/redouble/pass actions remain.

### 16.2 Trick-legality edge cases

- Following suit always outranks the option to trump.
- A player cannot discard while holding the led suit.
- No duty to beat a non-trump card when following a non-trump suit.
- When trump is led, raising is mandatory if possible.
- At All Trumps, raising in the led suit is mandatory if possible.
- At No Trumps, raising is never mandatory.
- Void player, partner winning in suit contract: any card legal.
- Void player, opponent winning, no trump yet: must trump if possible.
- Void player, opponent winning, trump already played, higher trump available: must overtrump.
- Void player, opponent winning, trump already played, no higher trump: any card legal in this profile.
- Off-suit cards at All Trumps do not beat the led suit.
- Off-suit cards at No Trumps never win.
- Trick winner, not previous seat order, determines the next leader.
- The last trick bonus is awarded once and only once.

### 16.3 Declaration edge cases

- Sequence rank is natural `7..A`, not trump rank.
- Sequence cannot wrap from A to 7.
- Sequence must be one suit.
- A run of five or more is one 100-point quinte.
- Maximal sequence must not be double-counted as smaller overlapping runs.
- Disjoint sequences may both be declared.
- Four 7s and four 8s are not declarations.
- Jacks beat nines in set comparison despite 200 vs 150 making this obvious.
- Among 100-point sets: A > 10 > K > Q.
- Same card in a sequence and a set requires a choice.
- Belot may overlap with a sequence.
- Exact sequence tie across teams cancels both teams' sequence category.
- Winning a declaration category awards all declarations in that category to the winning team.
- Losing sequence category does not cancel a won set category under this profile.
- No-Trumps ordinary declarations must be ignored/rejected.
- All Trumps may contain several belots.
- Belot declared after the first pair card was already played is late and invalid in manual mode.
- A declaration submitted after the player's first turn is rejected.
- An impossible declaration not supported by the hand is rejected server-side.
- Ordinary announcements belonging to a team that takes zero tricks are canceled in this profile.

### 16.4 Scoring edge cases

- Compare raw totals, never rounded totals.
- Add valid declaration points before deciding made/inside/hanging.
- Apply No-Trumps intrinsic ×2 before rounding.
- Do not intrinsically double the 90 capot bonus at No Trumps.
- Apply auction ×2/×4 after determining whole-deal scoreboard value.
- At ×2/×4, winner takes all current deal points.
- Existing hanging points are awarded to the next deal winner but are not multiplied again.
- At an undoubled tie, defender receives floor share; declarer share is ceiling-rounded and hangs.
- At a doubled tie, all multiplied current-deal points hang.
- An unequal raw score can display as equal after rounding; it is still not hanging.
- All-Trumps remainders 4/4 require the lower raw total to round up.
- No-Trumps base is 26, not 13.
- Suit capot is 25; All-Trumps and No-Trumps capot are 35 before declarations and auction multiplier.
- Inside transfers the whole current deal to defenders; declarer gets zero.
- Belot is not canceled by an opposing meld, but can end up in the all-or-nothing winner's award through contract settlement.

### 16.5 Match-end edge cases

- Score is checked only after a deal.
- Reaching 151 during a trick or declaration display does not immediately end the match.
- Both teams ≥151: higher total wins.
- Both teams ≥151 and equal: continue.
- Reaching 151 by capot: continue.
- Reaching 151 while points hang: continue.
- All-pass redeal does not clear capot finish lock.
- Another capot does not clear capot finish lock.
- A resolved non-capot, non-hanging deal clears the lock, then normal winner checks apply.

### 16.6 Persistence, reconnection, and concurrency edge cases

These are product rules, but they are essential to correct logic:

- Every action must carry a monotonically increasing `actionIndex` or state version.
- Duplicate submissions must be idempotent.
- Reject a stale action made against an old state version.
- Restore private hand data only to its owner after reconnect.
- Never send hidden opponent cards to a client and rely on the UI to conceal them.
- Persist auction multiplier reset events.
- Persist declared combinations and whether their timing was valid.
- Persist raw trick points; do not reconstruct solely from rounded score history.
- Persist hanging points and capot finish lock.
- Server timeouts should choose only from currently legal actions.
- If an AI replaces a disconnected player, it receives exactly the information available to that seat, not all hands.

---

## 17. UI behavior required by the rules

### 17.1 Table layout and direction

- Seat positions must make partners visually opposite.
- Turn animation must follow the selected direction profile.
- Dealer marker must be visible.
- First-player highlight must match the player to the dealer's right in this canonical profile.

### 17.2 Auction UI

Show:

- five-card hand only;
- legal contract buttons only;
- Pass, Double, and Redouble only when legal;
- current contract and multiplier;
- final bidder and declaring team;
- auction history in actual turn order.

After auction completion:

- animate/deal the remaining three cards;
- update hand sorting using the final contract;
- clearly distinguish bidder from first trick leader.

### 17.3 Card-play UI

- Compute legal cards on server and client.
- Disable or dim illegal cards.
- If a user taps an illegal card, explain the exact rule:
  - “You must follow clubs.”
  - “You must play a higher trump.”
  - “An opponent is winning; you must trump.”
- Do not use the vague error “Invalid move” when a specific reason is known.
- Show led suit and current winning card.
- At All Trumps, visually clarify that off-suit cards do not trump.
- At No Trumps, do not display trump indicators.

### 17.4 Declaration UI

- Before a player's first card is committed, show detected declaration options.
- If cards overlap between sequence and set plans, require a choice.
- Do not allow ordinary declarations at No Trumps.
- For Belot, prompt or automatically announce when the first K/Q card is played, depending on room settings.
- Ensure each belot scores once.
- Show declaration comparison outcome after all first plays:
  - accepted and scored;
  - lost to stronger opposing declaration;
  - canceled by exact tie;
  - invalid because of overlap or timing.

### 17.5 Score breakdown UI

Never display only one unexplained deal number. Show:

1. captured card points by team;
2. last-ten owner and +10;
3. No-Trumps intrinsic doubling, if applicable;
4. each valid declaration;
5. capot +90, if applicable;
6. raw totals used for comparison;
7. made / inside / hanging result;
8. rounding result;
9. auction multiplier;
10. hanging points collected or added;
11. final scoreboard awards;
12. cumulative match score.

This breakdown is the fastest way to diagnose logic disputes.

### 17.6 Match-end UI

If a team crosses 151 but cannot yet win, state why:

- “Reached 151 with capot — one more resolved non-capot deal is required.”
- “Points are hanging — play continues until they are resolved.”
- “Both teams are tied above 151 — play continues.”

Do not show a victory modal and then retract it.

---

## 18. Recommended rule-configuration switches

To support local or tournament variants without corrupting the core engine, isolate differences behind explicit settings:

```yaml
playDirection: COUNTER_CLOCKWISE          # alternative: CLOCKWISE
firstActorRelativeToDealer: RIGHT         # alternative: LEFT
bidOrder: [CLUBS, DIAMONDS, HEARTS, SPADES, NO_TRUMPS, ALL_TRUMPS]
passedPlayerMayReenterAuction: true
overtrumpWhenFollowingTrumpEvenIfPartnerWinning: true
mustTrumpWhenVoidAndPartnerWinning: false
mustUndertrumpWhenUnableToOvertrump: false
allTrumpsMustRaise: true
noTrumpsMustRaise: false
ordinaryDeclarationsAtNoTrumps: false
declarationCompetitionMode: SEPARATE_SEQUENCE_AND_SET
allowCardInSequenceAndSetSimultaneously: false
belotIndependent: true
cancelTricklessTeamOrdinaryDeclarationsOnCapot: true
noTrumpIntrinsicMultiplier: 2
noTrumpCapotBonusAlsoIntrinsicDoubled: false
counterMultiplierIncludesDeclarations: true
counterMultiplierIncludesCapot: true
counterCancelledByCapot: false
cannotFinishOnCapot: true
cannotFinishWithHangingPoints: true
targetScore: 151
suitRoundUpFromRemainder: 6
noTrumpRoundUpFromRemainder: 6
allTrumpRoundUpFromRemainder: 5
allTrumpFourFourCorrection: LOWER_TEAM_ROUNDS_UP
hangingDeclarerRounding: CEILING
hangingDefenderRounding: FLOOR
```

The selected values and `rulesProfileId` MUST be recorded with replays and server logs.

Notable alternative profiles may use:

- clockwise play and the player to dealer's left;
- one global comparison pool for sequences and sets;
- mandatory undertrumping;
- counter canceled by capot;
- different rounding convention;
- immediate win on capot;
- automatic declarations rather than manual declaration timing.

---

## 19. Suggested domain model

```typescript
type Suit = 'CLUBS' | 'DIAMONDS' | 'HEARTS' | 'SPADES';
type Rank = '7' | '8' | '9' | '10' | 'J' | 'Q' | 'K' | 'A';
type TeamId = 'A' | 'B';
type Seat = 0 | 1 | 2 | 3;

type Contract =
  | { type: 'SUIT'; suit: Suit }
  | { type: 'NO_TRUMPS' }
  | { type: 'ALL_TRUMPS' };

type AuctionAction =
  | { type: 'PASS'; seat: Seat }
  | { type: 'BID'; seat: Seat; contract: Contract }
  | { type: 'DOUBLE'; seat: Seat }
  | { type: 'REDOUBLE'; seat: Seat };

type Declaration =
  | { type: 'SEQUENCE'; seat: Seat; cards: CardId[]; length: number; topRank: Rank; points: 20|50|100 }
  | { type: 'FOUR_KIND'; seat: Seat; rank: Rank; cards: CardId[]; points: 100|150|200 }
  | { type: 'BELOT'; seat: Seat; suit: Suit; cards: [CardId, CardId]; points: 20 };

interface TrickPlay {
  seat: Seat;
  card: CardId;
  belotCall?: 'BELOT' | 'REBELOT';
}

interface Trick {
  leader: Seat;
  plays: TrickPlay[];
  winner?: Seat;
  rawCardPoints?: number;
}

interface DealScoreBreakdown {
  capturedCardPoints: Record<TeamId, number>;
  lastTen: Record<TeamId, number>;
  noTrumpIntrinsicFactor: 1 | 2;
  declarationPoints: Record<TeamId, number>;
  capotPoints: Record<TeamId, number>;
  rawTotals: Record<TeamId, number>;
  outcome: 'MADE' | 'INSIDE' | 'HANGING';
  auctionMultiplier: 1 | 2 | 4;
  currentDealAwards: Record<TeamId, number>;
  hangingBefore: number;
  hangingAwarded: Record<TeamId, number>;
  hangingAfter: number;
  cumulativeAfter: Record<TeamId, number>;
}
```

Prefer an append-only event log:

```text
DealStarted
CardsDealt
AuctionActionAccepted
AuctionCompleted
RemainingCardsDealt
DeclarationsSubmitted
CardPlayed
TrickCompleted
DeclarationsResolved
DealScored
MatchFinished
```

Derived UI state should be rebuildable from these events.

---

## 20. Validation invariants

Run these after every accepted action where applicable.

### 20.1 Deck and hand invariants

- Exactly 32 unique card IDs exist.
- Before play, every player has 8 cards.
- At trick number `t` before the current trick starts, every player has `8 - t` cards.
- Played cards plus cards in hands total 32.
- No card exists in two locations.

### 20.2 Turn invariants

- Exactly one active seat exists.
- Auction/play order follows the rules profile.
- A completed trick has exactly four plays from four distinct seats.
- Next leader equals previous trick winner.

### 20.3 Auction invariants

- Highest contract never decreases.
- Multiplier is only 1, 2, or 4.
- ×2 was created by a defending seat.
- ×4 followed ×2 and was created by the declaring team.
- A new higher contract resets multiplier to 1.
- Final declaring team equals team of final highest normal bidder.

### 20.4 Play invariants

- Accepted card belongs to the acting player.
- Accepted card is in `legalCards(...)`.
- Winner calculation uses the correct rank table.
- Sum of completed trick counts is eight per deal.

### 20.5 Point invariants

Without ordinary declarations/belot/capot:

- suit captured card points plus last ten = 162;
- No-Trumps transformed total = 260;
- All-Trumps captured card points plus last ten = 258.

Other checks:

- card point totals are based on captured tricks, not current hands;
- last ten is assigned exactly once;
- capot is true exactly when one team has eight trick wins;
- no team can receive capot bonus without eight tricks;
- No-Trumps ordinary declaration points are zero;
- raw totals are retained for outcome comparison;
- hanging pool never becomes negative;
- a non-tied settled deal clears existing hanging points;
- cumulative scores change only during deal settlement.

---

## 21. Minimum acceptance-test matrix

A serious implementation should include all of the following.

### 21.1 Auction tests

1. First legal bid from empty auction.
2. Reject equal bid.
3. Reject lower bid.
4. Permit re-entry after pass.
5. Four-pass redeal.
6. Three passes after contract.
7. Legal double by defender.
8. Reject double by declaring team.
9. Legal redouble by declaring team.
10. Reject redouble by defender.
11. Higher bid cancels double.
12. Higher bid cancels redouble.
13. Three passes after double.
14. Three passes after redouble.
15. All Trumps followed by valid double/pass options only.

### 21.2 Card-play tests

16. Lead accepts every card in hand.
17. Must follow suit.
18. Cannot trump while holding led suit.
19. Non-trump follow does not require raising.
20. Trump lead requires raising when possible.
21. Trump lead allows any trump when raising impossible.
22. Void + partner winning allows any card.
23. Void + opponent winning + no table trump requires trump.
24. Void + opponent winning + higher trump available requires overtrump.
25. Void + opponent winning + no higher trump allows discard under this profile.
26. All Trumps requires follow and raise.
27. All Trumps void player may discard any card.
28. No Trumps requires follow only.
29. Trick winner in suit contract with a ruff.
30. Trick winner in All Trumps ignores off-suit high card.
31. Trick winner in No Trumps ignores off-suit ace.

### 21.3 Declaration tests

32. Detect tierce.
33. Detect quarte.
34. Detect quinte of five.
35. Treat run of six as one quinte.
36. Detect two disjoint sequences.
37. Reject wraparound A-7 sequence.
38. Detect jack carré as 200.
39. Detect nine carré as 150.
40. Reject seven/eight carré.
41. Sequence length tie broken by top rank.
42. Exact sequence tie cancels sequence category.
43. Winning sequence team receives all own sequences.
44. Set and sequence categories resolve independently.
45. Overlap between set and sequence requires a plan.
46. Belot overlaps sequence legally.
47. Multiple All-Trumps belots score separately.
48. Reject ordinary declaration at No Trumps.
49. Reject late first-turn declaration.
50. Prevent duplicate belot scoring.

### 21.4 Scoring tests

51. Suit base total 162.
52. No-Trumps base total 260 after intrinsic doubling.
53. All-Trumps base total 258.
54. Suit made at 87–75 => 9–7.
55. Suit inside at 75–87 => 0–16.
56. Raw declaration points affect contract outcome.
57. All-Trumps 134–124 => display 13–13 but not hanging.
58. Undoubled 106–106 => defenders 10, hanging +11.
59. Doubled tie sends all doubled points to hanging.
60. Existing hanging awarded to next raw winner.
61. Existing hanging not multiplied by counter.
62. Double makes current deal all-or-nothing.
63. Redouble makes current deal all-or-nothing at ×4.
64. Suit capot base score 25.
65. No-Trumps capot base score 35.
66. All-Trumps capot base score 35.
67. No-Trumps capot bonus not intrinsically doubled.
68. Counter multiplier applies to capot in this profile.

### 21.5 Match-end tests

69. One team reaches 151 on normal resolved deal and wins.
70. Both teams exceed 151; higher wins.
71. Both teams equal above 151; continue.
72. Team reaches 151 by capot; continue.
73. All-pass does not clear capot lock.
74. Another capot does not clear capot lock.
75. Normal non-hanging deal clears capot lock.
76. Hanging pool blocks finish.
77. Resolving hanging pool allows finish check.

---

## 22. Property-based tests worth adding

1. **Legal-card non-emptiness:** for every valid state with a nonempty hand, `legalCards` is nonempty.
2. **Follow-suit property:** if hand contains led suit, every legal card has led suit.
3. **No hidden trump property:** in No Trumps and All Trumps, an off-suit card never wins.
4. **Card conservation:** every generated complete deal contains every card exactly once.
5. **Trick conservation:** eight completed tricks contain 32 distinct plays.
6. **Point conservation:** base raw totals match 162/260/258 before declarations and capot.
7. **Auction monotonicity:** normal contract rank only increases.
8. **Multiplier provenance:** ×4 is impossible without a prior valid ×2.
9. **Outcome precision:** changing only rounding must never change made/inside/hanging status.
10. **Hanging resolution:** after any unequal settled deal, hanging pool equals zero.
11. **Replay determinism:** replaying the same event log produces byte-identical public state and score breakdown.
12. **Seat symmetry:** rotating all seats while preserving teams and direction does not change legality or scores.

---

## 23. Recommended pure-function boundaries

Keep these deterministic and independently testable:

```text
contractRank(contract)
rankStrength(card, contract, ledSuit)
cardPointValue(card, contract)
legalAuctionActions(state, seat)
applyAuctionAction(state, action)
legalCards(hand, trick, contract, actorTeam)
determineTrickWinner(trick, contract)
detectDeclarationPlans(hand, contract)
resolveSequenceDeclarations(teamA, teamB)
resolveSetDeclarations(teamA, teamB)
resolveBelots(events, contract)
calculateRawTotals(tricks, declarations, contract)
classifyContract(rawTotals, declaringTeam)
roundTeam(raw, contract, bothRawTotals)
calculateWholeDealUnits(rawTotals, contract, capot)
settleDeal(...)
checkMatchEnd(...)
```

Do not mix UI state, animation state, timers, or network retry logic into these functions.

---

## 24. Common implementation failures and fixes

| Failure | Why it is wrong | Correct fix |
|---|---|---|
| One rank table for all contracts | J and 9 change strength/value | Select rank/value table per contract and suit role |
| Generic `Math.round` | Bulgarian thresholds and All-Trumps 4/4 case differ | Use explicit integer rounding functions |
| Compare displayed scores | 134–124 may display 13–13 | Compare raw totals |
| Force trump when partner wins | Not required in canonical suit contract | Check current winner's team first |
| Never force overtrump | Higher trump is mandatory in specified cases | Filter to higher trump cards |
| Let off-suit card win at All Trumps | “All trumps” changes rank, not lead-suit dominance | Winner still comes from led suit |
| Permanently remove passer from auction | Bulgarian auction permits re-entry | Keep passer active until auction ends |
| Keep double after higher bid | Double applies to the challenged contract | Reset multiplier on new contract |
| Allow melds at No Trumps | Canonical Bulgarian rules disable them | Return no ordinary declaration plans |
| Score every team's sequences | Only category-winning team scores sequences | Resolve category competition first |
| Let one card count in sequence and carré | Canonical profile forbids overlap | Require declaration plan |
| End immediately at 151 after capot | “No going out with capot” | Set finish lock and continue |
| Lose hanging points on redeal | All-pass does not resolve them | Preserve hanging pool |
| Multiply old hanging points on counter | Counter applies to current deal | Add old pool after current multiplier |
| Client-only legal moves | Modified client can corrupt game | Server validates every action |
| Send all hands to all clients | UI hiding is not information security | Server filters private state per seat |

---

## 25. Source and ambiguity notes

The canonical profile above is synthesized primarily from Bulgarian rule descriptions and cross-checked against scoring references:

- Belot.BG, **“Белот правила: раздаване, анонси и точкуване”**: https://belot.bg/belot/rules/
- Белот Треньор, **“Правила на белот: анонси и точкуване”**: https://belotcoaching.bg/pravila-na-belot/
- Белот Онлайн, **“Правила на белота — пълно ръководство”**: https://www.belota.bg/pravila.html
- Belotly, **“Belot Scoring Tables”**: https://www.belotly.com/belot-scoring/
- Belotly, **“How to Play Belot”**: https://www.belotly.com/how-to-play/
- Сметни, **“Белот: брояч на точки”**: https://smetni.bg/bg/tools/belot-counter/
- Example tournament rules showing local variants: https://cooldown.bg/belot_turning_pravila.pdf

### 25.1 Important implementation conclusion

There is no safe way to support every Bulgarian table convention with hard-coded behavior. Treat the values in Section 18 as part of saved game state. If an existing application's behavior differs from this document, first determine whether the bug is:

1. an actual violation of the selected profile;
2. an undocumented house-rule profile;
3. a UI display problem while server scoring is correct;
4. a raw-versus-rounded point confusion;
5. a replay/profile-version mismatch.

---

## 26. Final developer checklist

Before shipping, verify that the game:

- [ ] uses 32 unique cards and correct teams;
- [ ] uses the configured direction consistently for dealer, bidder, and play order;
- [ ] deals 3+2, auctions, then deals 3;
- [ ] implements full bid order and pass re-entry;
- [ ] validates double/redouble by team and resets on a higher bid;
- [ ] computes legal cards using current trick winner and contract;
- [ ] uses separate trump and non-trump rank/value tables;
- [ ] treats All Trumps as trump ranking in every suit without cross-suit trumping;
- [ ] disables declarations at No Trumps;
- [ ] resolves sequence, set, overlap, tie, and belot cases deterministically;
- [ ] retains raw totals until contract classification is complete;
- [ ] applies No-Trumps intrinsic doubling at the correct stage;
- [ ] implements Bulgarian rounding rather than generic rounding;
- [ ] handles inside, hanging, double, redouble, and accumulated hanging points;
- [ ] scores capot correctly and blocks immediate match exit;
- [ ] handles both teams reaching 151;
- [ ] gives users a full score breakdown;
- [ ] rejects illegal actions server-side;
- [ ] versions and persists the rules profile;
- [ ] passes every acceptance and property test listed above.

If all items pass, the rules engine and UI should agree on every important state transition and on the Bulgarian Belot edge cases most likely to cause production bugs.
