# Reflection

## 1. What did a fresh session get wrong or ask about that an earlier round had already settled? What was missing from the repository that would have prevented it?

In Round 2, the fresh session asked me what should happen if the player typed something
invalid at the new "Play again? (y/n)" prompt — should it quit, or ask again? But this was
already settled, in general form, back in Round 1's `CLAUDE.md` (commit `ff9d689`):
*"Never let invalid input crash the program — always re-prompt."* That rule isn't scoped
to the move prompt specifically — it's a standing rule for the whole program. The session
could have applied it directly to the new prompt instead of stopping to ask.

Nothing was actually missing from the repository — the rule was already there, stated
generally enough to cover this case. The session asked anyway, which I'd read as
reasonable caution rather than a real gap: a one-line rule in a file is a weaker signal
than an explicit "this applies to every prompt, including ones that don't exist yet," and
asking a genuinely ambiguous-sounding question costs one round-trip, while guessing wrong
on "does the game quit or continue" would have been a worse outcome to fix later.

## 2. In the round that changed an earlier requirement, what broke? Did the tests catch it, and does the decision record explain the change to someone who was not there?

Round 4 changed R3 (computer strategy: "a simple strategy" → "optimal play, never
loses"). What broke: the Round 1 test
`test_takes_center_when_open_and_no_win_or_block` — it asserted the old heuristic's
specific square choice, which a minimax search has no reason to reproduce. The tests
caught it immediately: Round 4's commit sequence was "Add failing tests for optimal
computer play (R3)" *before* "Make the computer play optimally with minimax (R3)" (commits
`4fd5507` then `2ff9839` in `tictactoe`), and the incompatible old test was explicitly
identified and replaced rather than silently left in place or deleted without comment
(`handoff-round-4.md`: *"Replaced `test_takes_center_when_open_and_no_win_or_block` (it
only checked the old strategy)"*).

Decision record `docs/decisions/0004-minimax-with-depth-and-lowest-square-tiebreak.md`
explains the change to someone who wasn't there: it states the context (R3's wording
change), the alternatives considered (extending the old rule list with a fork check, an
opening book, random tie-breaking) and why they were rejected (missed cases a full search
doesn't have, nondeterminism that would make "never loses" harder to test), not just what
was built.

## 3. Name one test that caught a real problem, or one test the assistant wrote that checked nothing useful. Quote it.

A test that checked nothing useful, from early in Round 1 (`tictactoe/test_tictactoe.py`,
before any fresh-session rounds started):

```python
def test_prefers_win_over_block(self):
    # O can win at 3 (positions 1,2) AND must otherwise block X's 7,8 -> 9.
    board = board_from("OOXX....X")
    # board: O O X X . . . . X  -> O wins by completing 1,2,3
    self.assertEqual(computer_move(board), 3)
```

This couldn't have passed against *any* correct implementation: position 3 (index 2) is
already occupied by `X` in that board string, so there is no winning move there for `O` to
find — the test was asserting an impossible outcome. Running it immediately failed
(`AssertionError: 6 != 3`), which is what actually caught the bug — not because the
*logic* was wrong, but because the *test's own setup* described a board that couldn't
exist under its own premise. It was rewritten as:

```python
def test_prefers_win_over_block(self):
    # O can win at position 3 (completing 1,2,3), while X separately
    # threatens to win at position 6 (completing 4,5,6). Win must come first.
    board = board_from("OO.XX....")
    self.assertEqual(computer_move(board), 3)
```

## 4. Choose one commit you would now write differently, and say how.

`tictactoe`, commit `f16a38b`: *"Implement undo (R10-R12); fix syntax error in the undo
tests."* The message itself admits two unrelated things landed in one commit: the real
undo feature, and a fix for a broken string literal in a test. Checking the diff
confirms where the error came from — the commit message says *"The previous commit's
test file had a broken string literal (a newline escape expanded by the shell); this
commit fixes it"* — so the mistake was already sitting in the prior commit, `3e4d84b`
("Add failing tests for undo"), not introduced fresh here.

What I'd do differently: split it into two commits, fix first. `3e4d84b` should never
have been committed with a syntax error in it at all — running the tests right after
writing them would have caught it immediately, before it became a committed mistake. But
given that it happened, the right repair is its own commit ("Fix syntax error in R10 undo
test"), separate from and before "Implement undo (R10-R12)" — not folded into the feature
commit's message as an afterthought. That keeps each commit doing exactly one thing, which
is what the assignment's "one change per commit" rule is actually for: a history where
every commit leaves the suite in a state you could bisect to.
