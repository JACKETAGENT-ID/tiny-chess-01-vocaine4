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
