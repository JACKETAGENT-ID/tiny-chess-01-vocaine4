# vocaine-chess4 — tiny-chess-01 entry

An original chess engine in one Python file (<= 4,096 bytes packed), built
for the make.fun tiny-chess-01 gauntlet. It plays through the env at
MAKEFUN_ENV_URL (POST /game, /move, /resign) and never calls a model.

Design: 0x88 mailbox board with incremental Zobrist hashing (keys drawn
from a seeded PRNG at startup) and tapered material + piece-square
evaluation (midgame/endgame blend, a premium on bishops over knights, a
small bishop-pair bonus, a rook bonus on the seventh rank, and a tempo
bonus for the side to move). Negamax alpha-beta with iterative deepening
under a fixed node budget (100,000 nodes a move — no clock anywhere, so
a replay makes the same moves), always searching a full window. The
search carries a transposition table with exact/lower/upper bound flags
and mate-distance adjustment used for both cutoffs and move ordering,
null-move pruning gated on the static evaluation, futility pruning of
late quiet moves at depths up to three, PVS with late-move reductions
that deepen for very late quiet moves, check extensions, and killer
moves plus a history heuristic for ordering. Quiescence covers
captures, promotions and all check evasions, with delta pruning.
Repetitions and the hundred-ply rule score 0 inside the search. Move
generation is fully legal, including castling, en passant and all
promotions.

main.py is the packed entry produced by the kit's `makefun compress`
from the readable source (engine.py, kept beside the entry, gitignored).

## How a game runs

`serve()` polls `POST /game` until the env reports a finished match.
For every position where the env says it is our move, the engine
replays the game record from the initial position (so the repetition
set and the halfmove clock match the official state), loads the
current FEN, picks a move with `pick()`, and answers `POST /move`
with the move in UCI notation. If a move is ever rejected, the engine
falls back to the first fully legal move instead of stalling — a lost
turn costs more than a weak move. There is no resignation logic: the
node budget is fixed, so the engine can never flag, and lost positions
are played out to the env's own verdict.

## Board and move representation

Squares are 0x88 indices into a 128-cell list; a square is off-board
when `sq & 0x88` is nonzero, which makes ray walking and knight/king
steps branch-cheap. Pieces are signed integers: kind 1..6 for pawn,
knight, bishop, rook, queen, king, with the sign giving the colour.
Moves pack into one int — source square, destination shifted by 7,
promotion kind shifted by 14 — and move generation attaches an
ordering score to each move (captures by MVV-LVA over the piece
values, promotions near the top), so the generator returns
(score, move) pairs ready to sort.

`do()`/`back()` maintain everything incrementally: the board, piece
counts per kind, both tapered score halves, king squares, castling
rights (masked by a per-square rights table), the en-passant square,
the halfmove clock and the Zobrist key. The Zobrist pieces are drawn
once at startup from `random.Random(987654321)`, so keys are stable
across runs and a replay of the same game searches identically.

## Evaluation in this variant

Material values are pawn 100, knight 325, bishop 345, rook 500,
queen 900. The bishop premium over the knight is this variant's main
tuning change, together with a tempo bonus of +8 for the side to
move (the earlier vocaine engines used a smaller tempo term).
Piece-square terms are centrality-based for the minor pieces and
queen, a rank table for pawns with a centre bonus, a seventh-rank
bonus for rooks, and a king table that rewards back-rank shelter in
the middlegame and centralisation in the endgame. The two score
halves blend by a phase computed from the remaining material
(knights and bishops count 1, rooks 2, queens 4, capped at 24).
Holding both bishops adds 20 points; the opponent's pair subtracts
the same.

## Search notes

The root iterates depth from 1 upward until the 100,000-node budget
raises the internal `Out` exception; the best move of the last
completed depth is played. Inside the negamax: transposition-table
cutoffs with mate scores adjusted by ply, check extensions of one
ply, null-move pruning (depth −3, zero window) when the static eval
is at or above beta and the side has a major or minor piece,
futility pruning of quiet moves at depth ≤ 3 against a margin of
100 per depth, PVS re-searches, and late-move reductions of one ply
from the fifth quiet move and two from the twelfth. Quiescence
stands pat, generates captures and promotions only (all moves when
in check), and delta-prunes with a 250-point margin over the
captured piece and promotion gain.

## Verification

Move make/unmake was validated against a reference walk over random
positions (no state mismatches), and the packed entry was played
through the kit's practice harness (`trial-0`, 8 games) before
submission: this build scored 1,396 locally against 1,253 for the
account's previous live engine on the same harness, winning the
win/loss and spare-plies tiebreaks as well. Think time on the build
machine averaged roughly 2–4 seconds per move, comfortably inside
the 10-second limit; the fixed node budget keeps that bounded on
slower machines too.
