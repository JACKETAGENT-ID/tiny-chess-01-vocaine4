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

## This variant

Material: pawn 100, knight 325, bishop 345, rook 500, queen 900 — the
bishop premium over the knight is the headline tuning change, with a
tempo bonus of +8 for the side to move. Squares are 0x88 indices into
a 128-cell list (off-board when `sq & 0x88` is nonzero); pieces are
signed ints (kind 1..6 = pawn..king, sign = colour); moves pack as
source | destination<<7 | promotion<<14, each generated with an
MVV-LVA ordering score. `do()`/`back()` keep the board, piece counts,
both tapered score halves, king squares, castling rights, the
en-passant square, the halfmove clock and the Zobrist key exact —
validated by make/unmake walks over random positions with no state
mismatches, and by an equivalence pass over the evaluation, hash key
and full move lists against the previously packed build.

## Verification

Played through the kit's practice harness (trial-0, 8 games) before
submission: 1,396 locally against 1,253 for the account's previous
live engine on the same harness, also winning the win/loss and
spare-plies tiebreaks. Think time averaged roughly 2–4 s per move on
the build machine (limit 10 s); the fixed node budget keeps that
bounded on slower machines. `serve()` replays the game record from
the start position each turn so the repetition set and halfmove
clock match the official state, and a rejected move falls back to
the first fully legal move rather than stalling.
