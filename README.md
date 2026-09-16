# chess.com tactics puzzles

952,898 tactics puzzles from chess.com containing position, solution, rating,
pass rate and solve-time stats for each one, in a JSON and a CSV format. You can
find all of the puzzles on the release page to download as a .zip file.

## Why

Chess.com gives free accounts 3 puzzles per day and puts the rest behind
Premium. The puzzles come from games people played, and the ratings come from
the people who solve them. I don't think that belongs behind a paywall, so I
collected them through the same API the puzzle pages use.

This release contains the rated tactics set only. The daily puzzles (which
chess.com publishes for free anyway, and can be found [here](https://github.com/samuraitruong/chess.com-daily-puzzle))
are not included, so every entry here has a numeric id, a rating, and a solution
in the same encoding.

## Files

Same content in two shapes:

| file | size | keyed by |
|---|---|---|
| `puzzles.json` | ~1.05 GB | puzzle id |
| `puzzles_by_fen.csv` | ~770 MB | position (`fen3`) |

Each is zipped in the release: `puzzles-json.zip` (~244 MB) and
`puzzles-csv.zip` (~220 MB).

CSV columns:

```
fen3,id,rating,initialFen,tcnMoveList,colorOfUser,pgn,passRate,averageSeconds,gameLiveId,gameId
```

A JSON entry looks like this:

```json
"1322993": {
  "id": 1322993,
  "fen3": "2r3k1/p4ppp/3p4/8/1P1qpP2/P6P/3B2P1/3Q1RK1 w -",
  "initialFen": "2r3k1/p4ppp/3p4/8/1P1qpP2/P6P/3B2P1/3Q1RK1 w - - 1 23",
  "tcnMoveList": "fnCunm6cdculgpBDowDnmnl{",
  "colorOfUser": "black",
  "rating": 3554,
  "passRate": 27.3,
  "averageSeconds": 85,
  "attemptCount": 1411,
  "gameLiveId": 12572304173,
  "gameId": null,
  "pgn": "[Event \"?\"]\n..."
}
```

Three things to know before parsing:

- `fen3` is the first three fields of a FEN: the board, side to move, and
  castling rights. No en passant square, no move counters. To look a position
  up, cut your FEN down to its first three fields
  (`fen3 = " ".join(fen.split()[:3])`).
- `tcnMoveList` is chess.com's TCN move encoding, not SAN or UCI. Two
  characters per move; decoder below. The full PGN is in the `pgn` column if
  you'd rather parse that instead.
- The `pgn` field contains real newlines. Any proper CSV parser handles the
  quoting; `split("\n")` does not.

## Decoding TCN

Each move is two characters: a from-square and a to-square, plus a promotion
piece when the to-character runs past the board. This turns a `tcnMoveList`
into a list of moves like `f1f2`, `e4e3`, ..., `d2c1q`:

```python
TCN = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!?{~}(^)[_]@#$"

def decode_tcn(tcn):
    moves = []
    for i in range(0, len(tcn), 2):
        a = TCN.index(tcn[i])
        b = TCN.index(tcn[i + 1])
        promo = ""
        if b > 63:                       # promotion: b encodes the piece + which file
            promo = "qnrbk"[(b - 64) // 3]
            b = a + (-8 if a < 16 else 8) + ((b - 64) % 3) - 1
        frm = TCN[a % 8] + str(a // 8 + 1)   # e.g. "f1"
        to = TCN[b % 8] + str(b // 8 + 1)
        moves.append(frm + to + promo)       # UCI-style, e.g. "d2c1q"
    return moves

# decode_tcn("fnCunm6cdculgpBDowDnmnl{")
# -> ['f1f2', 'e4e3', 'f2e2', 'c8c1', 'd1c1', 'e3d2',
#     'g1h2', 'd4f4', 'g2g3', 'f4f2', 'e2f2', 'd2c1q']
```

The moves alternate sides starting from `colorOfUser`'s opponent. The first
move is the setup move played *into* the puzzle position, then the solver's
reply, and so on. Feed them to any board library (python-chess, chess.js) from
`initialFen` to replay the line.

## Ratings

```
  100-299   #####                                           28,472
  300-499   #####                                           26,713
  500-699   #######                                         34,909
  700-899   ##############                                  74,795
  900-1099  ##################################            183,393
 1100-1299  ############################################## 246,645
 1300-1499  #######################                        121,833
 1500-1699  ##############                                  74,279
 1700-1899  ##########                                      52,386
 1900-2099  #########                                       48,157
 2100-2299  #####                                           27,856
 2300-2499  ###                                             13,968
 2500-2699  ##                                               8,517
 2700-2899  #                                                7,045
 2900-3099  #                                                2,674
 3100-3299  #                                                  849
 3300-3499  #                                                  300
 3500-3699  #                                                   87
 3700-4699  #                                                   20
```

952,898 puzzles, median 1206. Half of them are rated below 1200 and 97% below
2400, which is where most players are, so that's where the puzzles pile up.
Ratings shift as more people attempt a puzzle, so these are a snapshot from
September 2026.

Ratings run from 100 to 4410, but the highest handful are freshly generated
puzzles with few or no attempts yet, so those numbers are provisional. The
hardest puzzle that plenty of people have actually tried is rated 3935. If you
want only settled ratings, filter on `attemptCount`.
