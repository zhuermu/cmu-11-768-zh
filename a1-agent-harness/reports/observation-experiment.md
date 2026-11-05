# Observation experiment: board only vs. board + legal moves

The agent plays White against the server's deterministic bot. "Board only" runs with
`--omit-legal-moves`; "board + legal moves" adds the full legal-move list to every observation.
One game per model and condition.

**Model substitution.** The assignment pairs DeepSeek with `gpt-oss-120b`. We could not run it and used a different second model instead (name withheld here; called "the second model" below). DeepSeek ran as `deepseek-flash` (DeepSeek V4.1 Flash) on DeepSeek's own API.

| Model | Observation | `play_move` calls | Illegal | Illegal rate | Reached `game_over` | Result |
|---|---|---:|---:|---:|---|---|
| deepseek-flash | board only | 70 | 0 | 0.0% | yes (139 plies) | White wins |
| deepseek-flash | board + legal moves | 27 | 1 | 3.7% | yes (49 plies) | White wins |
| second model | board only | 63 | 0 | 0.0% | yes (123 plies) | White wins |
| second model | board + legal moves | 56 | 0 | 0.0% | yes (107 plies) | White wins |

Counting rules: every `play_move` the model emitted is a call. A call is illegal when the server
rejected the move (`... is not legal in the current position`). Transient `Could not reach the chess
server` errors (1 in DeepSeek with the list, 1 and 2 in the second model's games) are transport failures,
not rule violations, and are excluded. In each case the model resent the move on the next step and it
was accepted.

## Observations

- **Illegal moves were rare in every condition: 1 in 216 calls.** With one game per cell this sample
  cannot show that the legal-move list reduces illegal moves. The single illegal move happened *with*
  the list: DeepSeek played `d4d5` at move 11 while d4 held White's knight
  (FEN `r1bk1bnr/1ppn1ppp/8/p3q3/2BN4/2N1B3/PPP2PPP/R2Q1RK1 w - - 0 10`), moving it like a pawn even
  though `d4d5` was not in the list it had just been given.
- **The list costs tokens.** Per move, DeepSeek's completion grew from about 3,400 to about 4,400
  tokens with the list. For the second model the list raised the largest prompt from 18.9k to 22.7k tokens
  despite a shorter game (56 vs. 63 moves).
- **What fills the context depends on the model, not on the board.** DeepSeek's board-only game ended
  with a 257k-token prompt; the second model's ended at 18.9k after a similar number of moves. In DeepSeek's
  final prompt, 92% of the characters are its own `reasoning_content` from earlier turns, which our
  `build_prompt` sends back on every request; tool observations (the board) are 5%. The second model returns
  no reasoning text, so its history is mostly board observations (60%). Replaying reasoning is what made
  the DeepSeek games slow and expensive (7.6M prompt tokens for one game), and with a 131k-window model
  such as gpt-oss-120b it would have overflowed the context mid-game.
- **Game length varies for reasons outside the treatment.** The bot is deterministic, so different games
  come from different White moves; a 49-ply and a 139-ply win say more about the moves chosen than about
  the observation format.

## Operational note

The first DeepSeek attempt at the legal-moves game used the default `CHESS_TIMEOUT=1800`. Modal
reclaimed the sandbox after 30 minutes, 31 moves in, and every later `play_move` returned a connection
error until the 200-step limit. All four games above used `CHESS_TIMEOUT=3600`.
