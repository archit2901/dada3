# DADA3: A bug-fixing team

**Deployed app:** https://bug-fixing-team-hacn.onrender.com/

## Write-up

### Setup

- **Model:** `gemini-3.5-flash-lite` on the Gemini free tier, for every agent in both systems.
- **Sandbox:** Daytona, one fresh sandbox per verification for the team and one for each baseline run.
- **Settings:** `MAX_ROUNDS = 3` and `MAX_PARALLEL = 2`, as given.
- **Runs:** two of each system, all on 14 November 2026. Team run 1 is the one from notebook 02
  (`results/team.json`, `results/REPORT_team.md`). Baseline run 1 is `results/baseline.json`. Team
  run 2 and baseline run 2 come from the second-run cell in notebook 03 (`results/second_runs.json`).

I didn't change anything in the bundle: not the settings, not the prompts, and not the given code
in `bugteam/`. The one place my code differs a little from the steps is `after_tools` in the
fixer. It looks at all the tool results from the latest step, not only the last one, so it still
stops if the model calls `transfer_to_verifier` along with another tool in the same turn.

I did throw away an earlier attempt. I had a full set of runs from the night before, but one team
run stalled for about 20 hours (72,147 s) and lost issue #4 to a Daytona error, and one team run's
results file got overwritten and had to be rebuilt by hand. I didn't trust that set as a record,
so I started over and reran every notebook from the top. What's below is only the clean set. I
come back to the earlier runs once, in the comparison, because one of them changes how sure I can
be.

### Results

The numbers printed in section 2 of notebook 03:

| Run | Resolved | Triage | Bugs fixed | False claims | Calls | Tokens | Time |
|---|---|---|---|---|---|---|---|
| team | 9/10 | 10/10 | 6/7 | 0 | 94 | 2,039,866 | 501 s |
| baseline | 7/10 | 9/10 | 5/7 | 3 | 46 | 510,380 | 133 s |
| team (2) | 9/10 | 10/10 | 6/7 | 1 | 108 | 2,230,076 | 589 s |
| baseline (2) | 7/10 | 9/10 | 5/7 | 3 | 44 | 492,369 | 133 s |

Non-bugs left unchanged: 3/3 for both team runs, 2/3 for both baseline runs. The repository's own
tests passed after every run's fixes.

What each run got wrong, from section 4 of notebook 03:

- **team:** #6 only. Its pair crashed with `ReadError: [Errno 54] Connection reset by peer` before
  the verifier ever ran, so there's no fix and no claim. Every fix the verifier accepted passed the
  hidden tests.
- **team (2):** #2 only, and it's a false claim. The verifier accepted a `split_amount` fix that
  gets refunds wrong.
- **baseline, both runs:** the same three issues each time.
  - **#1:** it collapsed the hyphens but never stripped accents, so `"Crème Brûlée Mug"` still
    gives `crème-brûlée-mug`.
  - **#2:** it followed the reporter's suggestion and gave the leftover cent to the last person:
    `split_amount(1000, 3)` returns `[333, 333, 334]`, but the docstring says `[334, 333, 333]`. In
    run 2 the refund case is broken too: `split_amount(-1000, 3)` returns `[-334, -335, -335]`, which
    doesn't even add up to -1000.
  - **#3:** it called this a bug and changed `word_count` so that `"don't stop"` counts as 3 words,
    even though the docstring uses "don't" as its example of one word.

  All three were reported as fixed and checked, which is where its 3 false claims come from.

### Did the team do better?

In these runs, yes. The team resolved 9 of 10 issues both times and the baseline resolved 7 both
times. That's 20 points of the percentage resolved. Neither system moved at all between its own two
runs, so on this set the gap between the systems is bigger than the gap between two runs of the
same system. The false claims point the same way, and that's the measure I care about more: 0 and
1 for the team, 3 and 3 for the baseline.

I'm less sure than the table makes it look, for two reasons. First, two runs each is very little.
Second, in the set I threw away, the baseline had one run that resolved 9/10 and fixed all seven
bugs, and its other run resolved 7/10. So the baseline can swing by 2 issues between runs, which is
exactly the size of the gap I'm calling a win. Across all my runs, the team never made more than
one false claim, while the baseline made either zero or three. Apart from the run where a Daytona
error took out #4, the team resolved 9/10 every time, which the baseline matched only once. I
think the honest summary is that the team was usually better, and more consistently so on false
claims, but I can't rule out that some of the gap is luck.

Looking at where the baseline went wrong, I don't think the team's edge came from the handoff,
which never fired (see below). It came from two other parts of the design.

- **Triage.** It's one small call per issue that only asks whether the code matches the
  docstring, and it got all 10 right in both runs. The baseline got #3 wrong both times, then
  "fixed" it.
- **The verifier's instructions.** It's told that the docstring is the spec and to test every
  example in it. All three of the baseline's mistakes were cases where it went with what the
  reporter wanted instead of what the docstring says:
  - **#2:** it took the reporter's suggested fix.
  - **#3:** it took the reporter's expectation.
  - **#1:** it tested only the issue's two examples and never tried the accented title from the
    docstring.

The team's one false claim is the same kind of gap. In team (2), the #2 verifier's whole test was
one assertion from the issue, `split_amount(1000, 3) == [334, 333, 333]`. It skipped the refund
example in the docstring, `split_amount(-1000, 3) == [-334, -333, -333]`, even though its prompt
says to include every docstring example. The fix returns `[-333, -333, -334]` there, so a test
that followed the prompt would have caught it. "Verified" is only as strong as the test behind it,
and this one was weaker than it looked.

### The handoff

There were no handbacks in either team run. The cell in section 3 of notebook 03 printed nothing.
The logs show why: every pair's log is just "fixer -> verifier: ..." followed by "verifier
accepted the fix". That's all 13 verifications across the two runs (seven bugs, twice, minus the
#6 that crashed). For 12 of those 13, the hidden tests agree with the verifier, so these bugs are
mostly small enough that the fixer gets them right the first time. The 13th is the #2 fix in team
(2), which should have been sent back. It wasn't, because its test was too thin to fail. So the
handoff never got tested in the experiment: the verifier either had nothing to send back or didn't
notice when it did.

The only handback I saw was in notebook 01, when I ran the pair on #2 by itself, so I'll use that
one. The verifier sent the first fix back with this reason (shortened):

> The suggested fix in fixed/ implements the old docstring behavior (extra cent to the first
> shares), but the issue description states: 'Suggested fix: add the leftover cents to the last
> person's share...' Thus, fixed/ currently returns [-333, -333, -334] for split_amount(-1000, 3)
> instead of [-334, -333, -333] as specified in the docstring / issue description, or vice versa
> depending on whether last person or first person is intended.

By the docstring, it was half right. The refund complaint was correct: the docstring's example is
`[-334, -333, -333]`, and the first fix didn't give that. But the rest of the reason treats the
reporter's suggestion as if it competed with the docstring ("or vice versa depending on whether
last person or first person is intended"). The docstring is clear that the first shares get the
extra cent.

The next fix did not pass. The fixer switched to a different formula, which handled refunds but
moved the extra cent to the end: `split_amount(1000, 3)` returned `[333, 333, 334]` and
`split_amount(1, 3)` returned `[0, 0, 1]`, both against the docstring. The second verifier starts
fresh in a new sandbox. It wrote a test that expected `[333, 333, 334]`, changing the docstring's
own example to match. The test failed on the original code and passed on the fix, so the verifier
accepted it. The handoff carried useful information, but part of that information was wrong, and
nothing downstream caught it.

### What did it cost?

| | Team | Baseline | Team ÷ baseline |
|---|---|---|---|
| Model calls | 94, 108 | 46, 44 | about 2.2× |
| Tokens | 2.04M, 2.23M | 0.51M, 0.49M | about 4.3× |
| Time | 501 s, 589 s | 133 s, 133 s | about 4× |

Most of the team's cost is the verifier. Triage and the summary come to 11 calls per run, so the
other 83 to 97 calls are spread over seven pairs, roughly 12 to 14 calls per bug. Every one of
those calls carries the deep agent's long system prompt, which is why the token count goes up much
faster than the call count. On the free tier, one team run uses about a fifth of the day's 500
requests, against about a tenth for a baseline run.

For this repository, I think it was worth it. A false claim is the expensive outcome: the report
says a bug is fixed, someone merges it, and it ships. The team cut false claims from 3 per run to
0 or 1, and resolved two more issues per run, for about 50 to 65 extra calls and 1.5 to 1.7
million extra tokens. But that money didn't buy what I expected it to. The fixer-verifier loop,
the most expensive part, never sent a fix back. Most of the improvement seems to come from triage
(10 cheap calls) and from a verifier that tests against the docstring. If I were building this
for real, I'd try a cheaper version first: triage plus one verification pass per fix, without the
retry loop. I'd also want more than two runs of each before trusting any of these numbers.
