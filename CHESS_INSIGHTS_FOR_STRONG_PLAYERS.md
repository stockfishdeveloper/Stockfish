# Chess Insights from Stockfish Analysis
## Actionable Rules for 2200+ Rated Players

*Analysis based on Stockfish commit f66c3627 "Remove nmpColor" (April 5, 2023)*

This document extracts human-applicable chess principles from Stockfish's evaluation and search functions. While engines calculate precisely, these insights translate their logic into practical guidelines for strong human players.

---

## 1. Material Evaluation Insights

### Piece Values (Internal Stockfish Values)
- **Pawn**: 208 (base unit ~2.08 pawns = 1 pawn to humans)
- **Knight**: 781 (~3.75 pawns)
- **Bishop**: 825 (~3.97 pawns)
- **Rook**: 1276 (~6.13 pawns)
- **Queen**: 2538 (~12.2 pawns)

**Key Insight**: The bishop is worth slightly more than a knight (+44 centipawns, about 0.2 pawns). This small but consistent advantage suggests:
- **Rule 1**: In equal positions, prefer keeping bishops over knights unless specific positional factors (pawn structure, space) favor knights
- **Rule 2**: Two bishops are worth approximately 0.4 pawns more than bishop+knight or two knights

---

## 2. Position Evaluation and Material Complexity

### Static Evaluation Switch Point
From `evaluate.cpp`, Stockfish switches evaluation networks based on material imbalance:
```cpp
bool use_smallnet(const Position& pos) { 
    return std::abs(simple_eval(pos)) > 962; 
}
```

**Key Insight**: When material advantage exceeds ~4.6 pawns (962/208), position complexity changes fundamentally.

**Rule 3**: When you're up or down more than 4-5 pawns worth of material:
- Simplification becomes critical for the winning side
- The losing side should avoid trades and seek complications
- Positional factors matter less; material dominates

### Rule50 Dampening
Evaluation is reduced linearly by the 50-move rule counter:
```cpp
v -= v * pos.rule50_count() / 212;
```

**Rule 4**: As you approach the 50-move rule:
- Every ~21 moves without pawn moves or captures reduces evaluation by ~10%
- If winning, make pawn moves or captures before move 40-45 to reset the counter
- If defending, avoid unnecessary pawn moves to run down the clock

---

## 3. Search Depth and Move Ordering

### Null Move Pruning Insights
From search.cpp, null move reduction formula:
```cpp
Depth R = 7 + depth / 3;
```

**Key Insight**: When you have the advantage, giving your opponent a free move still allows you to maintain advantage if your position is strong enough.

**Rule 5**: If you can pass your turn and still maintain your advantage:
- Your position has deep strategic strength, not just tactical threats
- Focus on improving pieces rather than forcing immediate tactics
- Your opponent likely has no good counterplay

### Verification Search Threshold
```cpp
if (nmpMinPly || depth < 16)
    return nullValue;
```

**Rule 6**: Positions requiring 16+ ply (8 full moves) of verification are critically sharp:
- When both sides have active pieces and threats
- Mutual hanging pieces or forcing sequences exist
- Calculate deeper in these positions before committing

---

## 4. Late Move Reductions (LMR) - Move Quality Assessment

### Futility Move Count
```cpp
if (moveCount >= (3 + depth * depth) / (2 - improving))
    mp.skip_quiet_moves();
```

**Key Insight**: The number of reasonable moves in a position is roughly `(3 + depth²) / 2`.

**Rule 7**: In typical middlegame positions (depth ~6-8):
- Consider 20-25 candidate moves maximum when improving
- Consider 40-50 candidate moves when not improving (defense requires more options)
- Beyond this, moves are likely too weak to matter

### Improving Position Indicator
```cpp
improving = ss->staticEval > (ss - 2)->staticEval;
```

**Rule 8**: Compare your current position to 2 moves ago (one full move):
- If improving: Play more aggressively, try tactics, reduce opponent options
- If not improving: Calculate more carefully, seek counterplay, maintain flexibility
- This is more important than comparing to the previous half-move

---

## 5. King Safety and Attack Principles

### Null Move and King Safety
Null move is only attempted with material on board:
```cpp
if (cutNode && ss->staticEval >= beta - 18 * depth + 390 
    && !excludedMove && pos.non_pawn_material(us) 
    && ss->ply >= nmpMinPly && !is_loss(beta))
```

**Rule 9**: You need pieces (not just pawns) to maintain threats:
- Don't trade all your pieces if you want to maintain pressure
- Even with a pawn advantage, no pieces means no winning chances in many positions
- Keep at least one minor piece to maintain threat potential

---

## 6. Capture Evaluation and Exchanges

### Static Exchange Evaluation (SEE) Thresholds

**Captures in main search** (depth-dependent):
```cpp
int margin = std::max(157 * depth + captHist / 29, 0);
if (!pos.see_ge(move, -margin))
    continue;
```

**Key Insight**: You can sacrifice up to ~157 centipawns (0.75 pawns) per ply of depth for a capture.

**Rule 10**: When calculating sacrifices:
- A 1-ply deep combination can sacrifice ~0.75 pawns worth of material
- A 2-ply combination can sacrifice ~1.5 pawns
- A 3-ply combination can sacrifice ~2.25 pawns
- If your calculation is deeper, larger sacrifices are justified

**Quiet moves SEE threshold**:
```cpp
if (!pos.see_ge(move, -27 * lmrDepth * lmrDepth))
    continue;
```

**Rule 11**: For quiet moves (non-captures), acceptable losses scale quadratically:
- Never leave pieces hanging for "nothing" (tactical shots are bad trades)
- Positional sacrifices must lead to concrete plans within 2-3 moves

### Quiescence Search Capture Threshold
```cpp
if (!pos.see_ge(move, -78))
    continue;
```

**Rule 12**: In tactical sequences, don't capture if you lose more than ~0.37 pawns (78/208):
- Avoid "desperado" captures that lose material
- In mutual hanging piece positions, calculate precisely
- The side capturing first often has the advantage in equal exchanges

---

## 7. Tactical Pattern Recognition

### Singular Move Extension
```cpp
Value singularBeta = ttData.value - (56 + 81 * (ss->ttPv && !PvNode)) * depth / 60;
```

**Key Insight**: A move is "singular" (forced) when it's 1-2 pawns better than alternatives.

**Rule 13**: If one move is clearly best (1.5+ pawns better than others):
- This move is likely objectively forced
- Look for tactics that make this move necessary
- Opponent's move was likely an error if it gives you such a clear best move
- If you're choosing between similar moves (within 0.5 pawns), position is complex

### Multi-cut Pruning Insight
```cpp
else if (value >= beta && !is_decisive(value))
{
    ttMoveHistory << std::max(-400 - 100 * depth, -4000);
    return value;
}
```

**Rule 14**: When multiple moves seem to fail high (beat the expected score):
- The position is tactically unstable for the opponent
- You have multiple good options (initiative)
- Push your advantage quickly; don't let opponent consolidate
- Opponent's last move likely created multiple weaknesses

---

## 8. Pawn Structure and Endgame Principles

### Futility Margins
```cpp
Value futilityValue = ss->staticEval + 231 + 211 * lmrDepth
                    + PieceValue[capturedPiece] + 130 * captHist / 1024;
```

**Rule 15**: Material gains from captures are amplified in better positions:
- Base value of winning material: Static eval + 231 centipawns (~1.1 pawns)
- With good position, even small material gains (pawns) become decisive
- In worse positions, need larger material gains to equalize

### Stalemate Detection in Endgames
```cpp
if (!ss->inCheck && !moveCount && !pos.non_pawn_material(us)
    && type_of(pos.captured_piece()) >= ROOK)
{
    // Check for stalemate possibility
}
```

**Rule 16**: When you've just captured a rook or queen and only have pawns left:
- **Immediately** check for stalemate tricks
- Ensure you have legal pawn moves or king moves
- This is a critical moment where careless play can throw away a win

---

## 9. Time Management Concepts

### Best Move Stability
```cpp
double timeReduction = 0.723 + 0.79 / (1.104 + std::exp(-k * (completedDepth - center)));
```

**Rule 17**: If your best move stays the same over multiple candidate considerations:
- Spend less time; the move is clearly best
- Typical time reduction: 30-50% when move is obvious
- Spend more time when frequently changing your mind (move instability)

### Falling Evaluation Factor
```cpp
double fallingEval = (11.325 + 2.115 * (prevScore - bestValue)) / 100.0;
```

**Rule 18**: When your evaluation is dropping (position worsening):
- Spend MORE time (up to ~50% more)
- You're likely missing something or facing a strong plan
- Reconsider your previous move's quality
- Look for tactics you may have overlooked

---

## 10. Opening and Piece Development

### Low Ply History
```cpp
lowPlyHistory.fill(97);
```

**Key Insight**: Early in the game, moves have baseline value that gets refined.

**Rule 19**: In the opening (first 10-15 moves):
- All reasonable developing moves have similar value initially (~0.4-0.5 pawns)
- Don't spend excessive time on move 5-10 unless there's a concrete tactic
- Early position evaluations have higher uncertainty
- Stick to principles: development, king safety, center control

---

## 11. Correction History and Position Understanding

### Evaluation Correction by Position Type
```cpp
const auto pcv   = w.pawnCorrectionHistory[pawn_correction_history_index(pos)][us];
const auto micv  = w.minorPieceCorrectionHistory[minor_piece_index(pos)][us];
```

**Rule 20**: Position types have characteristic evaluation adjustments:
- Pawn structures have persistent positional value
- Minor piece positions (bishop vs knight) depend heavily on pawn structure
- Learn typical pawn structures and their evaluations
- Closed positions: Knights > Bishops (typically +0.3-0.5 pawns)
- Open positions: Bishops > Knights (typically +0.3-0.5 pawns)

---

## 12. Aspiration Windows and Confidence

### Aspiration Window Sizing
```cpp
delta = 5 + threadIdx % 8 + std::abs(rootMoves[pvIdx].meanSquaredScore) / 9000;
Value avg = rootMoves[pvIdx].averageScore;
alpha = std::max(avg - delta, -VALUE_INFINITE);
beta  = std::min(avg + delta, VALUE_INFINITE);
```

**Rule 21**: Evaluation confidence relates to position volatility:
- Stable positions: Evaluation changes by ~0.05 pawns between depths
- Unstable positions: Evaluation swings by 1+ pawns
- In unstable positions, calculate concrete variations to the end
- Don't rely on general principles in tactical melees

### Optimism Factor
```cpp
optimism[us] = 137 * avg / (std::abs(avg) + 91);
```

**Rule 22**: When ahead, you can afford to be "optimistic" about unclear positions:
- Up material: Play for concrete advantage, accept slightly worse positions
- Down material: Must play precisely, can't afford to give up more
- This is why "desperado" tactics work better when already losing

---

## 13. Continuation History - Follow-up Moves

### Move Pair Correlation
```cpp
(*(ss - 2)->continuationCorrectionHistory)[pc][to] << bonus * 137 / 128;
(*(ss - 4)->continuationCorrectionHistory)[pc][to] << bonus * 64 / 128;
```

**Rule 23**: Move sequences matter more than individual moves:
- A move 2 plies ago (ss-2) has 137% correlation with current position quality
- A move 4 plies ago (ss-4) has 64% correlation
- Look for move sequences that work well together
- After your opponent's move, consider: "What was the point 2 moves ago?"

---

## 14. ProbCut - Proving Moves are Good/Bad Quickly

### ProbCut Beta Margin
```cpp
probCutBeta = beta + 224 - 64 * improving;
```

**Rule 24**: A capture that looks ~1 pawn better than needed (224cp ≈ 1.08 pawns):
- Is very likely to be actually good at full depth
- If you have such a capture, opponent's position is likely already losing
- Conversely, if you're defending and opponent has such a capture, be very worried

---

## 15. Depth-Based Strategic Thinking

### Razoring Threshold
```cpp
if (!PvNode && eval < alpha - 514 - 294 * depth * depth)
    return qsearch<NonPV>(pos, ss, alpha, beta);
```

**Rule 25**: Position evaluation has depth-dependent thresholds:
- At depth 1: Can skip if down 808 centipawns (~4 pawns)
- At depth 2: Can skip if down 1690 centipawns (~8 pawns)
- At depth 3: Can skip if down 3158 centipawns (~15 pawns)

**Practical Application**: When significantly behind in material:
- Need immediate concrete tactics, not positional play
- If no tactics exist in 2-3 moves, position is likely lost
- Don't play on hoping for opponent errors in clearly lost positions

---

## 16. History Heuristics - Learning from Experience

### Main History Bonus/Penalty
```cpp
update_quiet_histories(pos, ss, *this, ttData.move,
                      std::min(130 * depth - 71, 1043));
```

**Rule 26**: Move quality memory from similar positions:
- Good moves from depth 8: bonus of ~969 points
- Similar positions suggest similar good moves
- If a move type worked well recently, consider it first
- Pattern recognition: "moves like this" matter more than specific moves

---

## 17. Cut Node vs All Node Strategy

### Cut Node Behavior
```cpp
if (cutNode)
    r += 3094 + 1056 * !ttData.move;
```

**Rule 27**: In "cut node" positions (expected opponent to beat target score):
- These are opponent's critical defensive positions
- They will find the best defense
- Don't assume best-case scenarios
- Calculate opponent's best replies more carefully

---

## 18. Decisive Scores and Practical Play

### Avoiding Decisive Score Ranges
Multiple places avoid "decisive" scores (tablebase range):
```cpp
if (!is_decisive(ttData.value))
if (!is_loss(bestValue))
if (!is_win(eval))
```

**Rule 28**: Near-mate or tablebase positions behave differently:
- When clearly winning (mate in N), play most forcing moves
- When clearly losing, play moves that maximize opponent's calculation burden
- In "practical" ranges (±5-8 pawns), normal rules apply
- Beyond ±8 pawns, position is essentially decided

---

## 19. Move Count and Position Complexity

### Early Move Benefits
```cpp
r += 843;  // Base reduction offset
r -= moveCount * 66;  // Bonus for early moves
```

**Rule 29**: First few moves in a position are disproportionately important:
- Move 1: Gets significant attention
- Move 2-3: Still heavily analyzed
- Move 4-10: Progressively less valuable
- Move 11+: Only analyzed if they look promising

**Practical Application**:
- In critical positions, make sure your first 3 candidate moves are actually the right ones to consider
- Don't get fixated on a first idea that seems good; check alternatives
- If your "best move" keeps changing, position is complex—calculate more

---

## 20. Capture History Impact

### Capture Quality Learning
```cpp
ss->statScore = 803 * int(PieceValue[pos.captured_piece()]) / 128
              + captureHistory[movedPiece][move.to_sq()][type_of(pos.captured_piece())];
```

**Rule 30**: Not all captures of equal material are equally good:
- Capturing with certain pieces to certain squares is better/worse
- Example: BxN on f6 might be better than BxN on c6 based on typical continuations
- Learn which piece trades are favorable in your typical positions
- A "good" capture can be worth up to 0.5 pawns more than a "bad" capture of same material

---

## Summary: Top 10 Most Actionable Rules

For quick reference, here are the most immediately applicable rules for strong players:

1. **Bishop > Knight by ~0.2 pawns** - Prefer bishops in equal positions
2. **Improving = Good** - If your position is better than 2 moves ago, play aggressively
3. **1+ pawn sacrifice per ply** - You can sacrifice ~0.75 pawns per ply of combination depth
4. **Singular moves are 1.5+ pawns better** - If one move is clearly best, it's likely forced
5. **Stay improving** - Compare position to 2 moves ago, not just last move
6. **Material > 4-5 pawns** - At this threshold, complexity changes; simplify if winning
7. **Stalemate check after Rook/Queen capture** - Critical defensive resource when down to pawns only
8. **Falling eval = think longer** - If position deteriorating, something is wrong
9. **First 3 candidates matter most** - Make sure you're considering the right moves initially
10. **Move sequences > single moves** - Look back 2 moves to understand current position

---

## Conclusion

Stockfish's evaluation and search functions reveal deep chess truths that strong players can apply:

- **Material imbalances** have precise thresholds where strategy changes
- **Move sequences** and position improvement over time matter more than single moves
- **Calculation depth** directly relates to allowable sacrifices
- **Pattern recognition** through history heuristics mirrors human learning
- **King safety** requires maintaining pieces for threats
- **Endgame technique** includes awareness of stalemate tactics
- **Time allocation** should depend on position stability and evaluation trends

These insights translate engine precision into human understanding, providing concrete guidelines for 2200+ players to improve their practical play and positional understanding.

---

*Document created from analysis of Stockfish source code at commit f66c3627*
*Focus areas: search.cpp, evaluate.cpp, types.h, movepick.cpp*
*Analysis date: November 16, 2025*
