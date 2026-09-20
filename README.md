# regexbench

**Evaluate a regex the way a benchmark should.**

Scoring generated regular expressions by string comparison is wrong: `(ab)+`
and `ab(ab)*` are the same language and different strings. And a pattern that
passes every test can still hang a production server.

`regexbench` answers the three questions that actually matter — is it the same
language, does it behave, and is it safe to run.

## Semantic equivalence

Both patterns compile to DFAs and the automata are compared, which is the
DFA-EQ metric used in the regex generation literature.

```python
from regexbench import equivalent

bool(equivalent(r"(ab)+", r"ab(ab)*"))      # True
bool(equivalent(r"[0-9]+", r"[0-9][0-9]*"))  # True

result = equivalent(r"a+", r"a*")
result.verdict     # <Verdict.DIFFERENT>
result.witness     # '' — the shortest string telling them apart
```

Witnesses are shortest-first and real: every one is a string you can paste
into `re.fullmatch` to see the difference yourself.

**When it says it doesn't know.** Backreferences make a pattern non-regular,
and equivalence is then formally undecidable. Rather than guess, the verdict is
`UNDECIDABLE`:

```python
equivalent(r"(a)\1", r"aa").verdict     # <Verdict.UNDECIDABLE>
equivalent(r"(?=a)ab", r"ab").verdict   # <Verdict.EQUIVALENT>
```

The two are kept apart on purpose. Lookaround is regular — it only escapes the
regular languages when combined with backreferences — so it is decided exactly
(lookahead and fixed-width lookbehind), and `UNDECIDABLE` is reserved for
patterns where nothing can answer.

**Shorthand classes follow `re`, which means Unicode.** `\d` matches every
Unicode digit, so it is not `[0-9]`:

```python
equivalent(r"\d", "[0-9]").verdict     # <Verdict.DIFFERENT>
equivalent(r"\d", "[0-9]").witness     # '٣'
```

That is pedantic and it is also what `re` does — and `check()` runs the real
`re`, so an engine that called them equivalent would contradict the tool it
lives in.

## Match semantics

Equivalence and scoring default to **full matches**, like `re.fullmatch`. Not
every benchmark means that, and the disagreement is silent — a reference
written for `re.search` simply looks wrong when full-matched.

```python
from regexbench import Semantics, equivalent

equivalent("a", ".*a.*").verdict                              # DIFFERENT
equivalent("a", ".*a.*", semantics=Semantics.SEARCH).verdict  # EQUIVALENT
```

Under `SEARCH`, `p` is rewritten to `.*p.*` around whatever it already
anchors, so there is still exactly one notion of equivalence underneath. The
wildcards distribute over a top-level alternation, because in `a|b$` the
anchor constrains `b` and says nothing about `a`.

Measured on Re(gEx|DoS)Eval: its reference expressions pass **100%** of their
own tests under `search` and **94.0%** under `fullmatch`. Choosing wrong there
would score 46 gold patterns as failing the tests they were written for.

## Dialects

The natural-language-to-regex corpora are not written in Python syntax. They
use `dk.brics.automaton` notation, where `&` is intersection and `~` is
complement — and Python's `re` compiles both as ordinary literals without
complaint.

```python
from regexbench import Dialect, equivalent

# In Python this is the literal "a&b". In dk.brics it is the empty language.
equivalent("(a)&(b)", r"a\&b").verdict                        # EQUIVALENT
equivalent("(a)&(b)", r"a\&b", dialect=Dialect.BRICS).verdict # DIFFERENT

equivalent("([0-9])&([0-4])", "[0-4]", dialect=Dialect.BRICS).verdict  # EQUIVALENT
```

`&` appears in 22.8% of KB13 and 27.3% of NL-RX, `~` in 7.6% and 17.2%. The
dialect is never sniffed, because both readings compile and only one is right.

The grammar follows
[dk.brics.automaton](https://www.brics.dk/automaton/doc/dk/brics/automaton/RegExp.html)
exactly, including precedence — union binds loosest, then intersection, then
concatenation, then repetition, then complement. Note that equivalence with
*both* complement and intersection is
[non-elementary](https://www.cs.umd.edu/~gasarch/TOPICS/desc/regexpcompint.pdf),
not merely PSPACE-complete: deeply nested `&`/`~` is refused with
`UNSUPPORTED` once determinization passes a state budget, rather than run until
the machine gives up.

## ReDoS safety

Two passes. Structural analysis finds the shapes that backtrack
catastrophically and says *why*; an empirical pass then runs the pattern
against attack strings under a timeout to catch what the structural pass
doesn't model.

```python
from regexbench import screen

screen(r"(a+)+").risk        # <Risk.EXPONENTIAL>
screen(r"(a+)+").reason      # 'a quantifier wraps a quantified group...'
screen(r"\d{3}-\d{4}").risk  # <Risk.SAFE>
```

`SAFE` means "no known-bad shape and no blow-up on what we tried". That is a
screening result, not a proof.

The structural pass covers three of the five vulnerability families named in
[the ICPC 2024 study of LLM-generated regexes](https://dl.acm.org/doi/10.1145/3643916.3644424):
nested quantifiers, exponential overlapping disjunction, and polynomial
overlapping adjacency. Exponential overlapping adjacency and starting-with-large-quantifier
are not modelled structurally and are only caught when the empirical pass
happens to trip them. That study also found LLM-generated regexes skew toward
*polynomial* ReDoS — the cheaper family to miss, and the one a short attack
string is least likely to expose.

## Running untrusted patterns

Python's `re` has no timeout, and a pathological pattern spins inside a single
C call that no signal or thread can interrupt. The only reliable escape is a
separate process:

```python
from regexbench import safe_search, MatchTimeout

try:
    safe_search(r"(a+)+$", "a" * 40 + "!", timeout=0.5)
except MatchTimeout:
    ...   # the process was killed; your server is still up
```

Costs milliseconds per call. Use it for model output and user input; use `re`
directly for patterns you wrote.

## Scoring against examples

```python
from regexbench import Task, evaluate

task = Task(
    prompt="three digits, a hyphen, four digits",
    positives=["123-4567"],
    negatives=["123-456", "abc"],
    reference=r"[0-9]{3}-[0-9]{4}",
)

report = evaluate(r"[0-9][0-9][0-9]-[0-9]{4}", task)
report.correctness.accuracy   # 1.0
report.equivalence.verdict    # <Verdict.EQUIVALENT>
report.usable                 # True — correct *and* safe
```

`usable` is the one to gate on: never a ReDoS liability, never *proven*
different from the reference, and perfect on whatever examples exist. A pattern
that passes every example it was given can still be known-wrong —
`#[0-9a-f]{6}` passes a hex-colour task whose examples happen to be lowercase —
and the reference settles it.

## Benchmarks

Scores are comparable only when they are computed on the same problems, so
loaders are included for the corpora this literature reports on. No dataset is
redistributed; you download the files and pass the path.

```python
from regexbench.datasets import load_regexeval, load_deep_regex, load_tasks

tasks = load_regexeval("RegexEval.json")      # 762 real prompts, with tests
tasks = load_deep_regex("datasets/KB13")      # 824 gold patterns, no examples
tasks = load_tasks("my_eval.jsonl")           # your own
```

Each loader sets the semantics and dialect the corpus actually uses. Loaders
never filter: a record this engine cannot represent is still returned and
surfaces as `UNSUPPORTED` when scored, because a corpus quietly reduced to its
easy half reports a number nobody can interpret.

See [docs/benchmarks.md](docs/benchmarks.md) for where to download each one and
what coverage to expect.

## Checking the engine itself

Everything above compares two *patterns*. `crosscheck` compares one pattern's
automaton to `re`, string by string, which is the sharper question — and the
one that finds bugs in this package:

```python
from regexbench import crosscheck

crosscheck(r"(ab)+").agreement       # <Agreement.AGREES>
crosscheck(r"(a)\1").agreement       # <Agreement.UNCHECKED> — a stated refusal
crosscheck(r"(a)\1").reason          # 'backreferences make the language non-regular'
```

A *verdict* about a pair only goes wrong when the two patterns go wrong in
different ways, so a mistake this engine makes uniformly cancels out of it.
Membership has no such cancellation. Point it at a file of patterns:

```bash
regexbench crosscheck uniq-regexes-8.json --registry pypi
```

That is how the last five wrong-answer bugs here were found, on corpora of
regexes people actually wrote rather than on anything curated. If you are
deciding whether to trust a verdict from this package, run it on your own
patterns.

## Scoring a whole model

```python
from regexbench import run

report = run(tasks, predictions, name="my-model", workers=8)
print(report.table(ks=(1, 5)))
```

```
my-model
762 tasks, 762 answered
  pass@1               77.0%
  dfa-eq@1             56.2%  (whole corpus — a lower bound)
  dfa-eq@1 (decided)   74.9%  (engine limits excluded — model only)
  exact@1              55.4%
  usable@1             65.4%
  vulnerable@1         12.1%  (lower is better)
  180 task(s) undecidable — counted against dfa-eq, excluded from dfa-eq (decided)
```

`predictions` is a mapping from task name to the pattern, or to a list of
sampled patterns, or a sequence aligned with the tasks. All metrics use the
unbiased pass@k estimator, so they line up with published numbers.

**A metric no task can answer is `None`, not zero** — KB13 ships no examples,
and a 0% pass@1 would read as a model failing a question nobody asked it.

**`dfa-eq` is reported twice**, because one number cannot answer both honest
questions. The plain figure counts undecidable comparisons as failures: how
much of the corpus was *verified* correct, a lower bound that cannot flatter.
The `(decided)` figure drops those tasks from the denominator: how much of what
could be checked was correct, the model alone. On Re(gEx|DoS)Eval the spread is
the engine's coverage: 86.6% of its references parse under the search
semantics it is scored with, so on the other 13.4% every candidate that is not
textually identical comes back undecidable and scores zero under the first
reading. (KB13 used to be the example here, when word boundaries were refused;
all three dk.brics corpora parse in full now.)

## CLI

```bash
regexbench eq '[0-9]+' '[0-9][0-9]*'        # equivalent
regexbench eq --search 'a' '.*a.*'          # equivalent
regexbench eq --brics '([0-9])&([0-4])' '[0-4]'
regexbench safety '(a+)+'                   # exponential
regexbench check '\d{3}' task.json          # 5/5 (100%)

regexbench run regexeval RegexEval.json --predictions preds.json --k 1 5
```

Exit codes are meaningful, so these compose in CI: `0` on equivalent/safe/all
passing, `1` otherwise.

`run` also takes `--use-reference`, which scores every task against its own
gold answer. That sounds circular and is the cheapest sanity check available:
if `pass@1` comes back well below 100%, the dataset is loaded with the wrong
match semantics and every later number is meaningless.

## Supported syntax

The equivalence engine covers the genuinely regular subset: literals, escapes
(`\d \w \s` and negations, plus `\xHH`, `\uHHHH`, `\UHHHHHHHH`, `\N{NAME}`,
`\a` and octal), `.`, character classes with ranges and negation, `*` `+` `?`
`{m,n}`, alternation, and grouping. In the `BRICS` dialect it also
covers intersection (`&`), complement (`~`), any-string (`@`) and the empty
language (`#`) — all regular operations, computed on the automata directly.
Anything else returns `UNSUPPORTED` or `UNDECIDABLE` rather than a wrong
answer.

Anchors are resolved wherever they appear, not just at the ends. Under
full-match semantics `^` can only hold where everything before it is empty, so
`a^` is the empty language, `a?^c` is `c`, and `(^a)*` is `a?` — the same
strings Python matches. Under `SEARCH` semantics an anchor away from the ends
is refused instead: the `.*p.*` rewrite cannot express it.

`$` is not end-of-string. Without `re.MULTILINE`, Python's `$` also matches
immediately before a newline that ends the subject, so `re.search(r"b$", "b\n")`
finds a match and `re.fullmatch(r"a$\n", "a\n")` is not the empty language.
Under `SEARCH` the reduction allows exactly that one trailing newline. Under
`FULLMATCH` there is nothing to widen, so a `$` sitting in front of text that
could be that newline is refused; a `$` at the end of the pattern is decided as
usual.

Patterns Python's own parser rejects are rejected here too — `a**`, `\b*`,
`\q`, `[\d-z]`. A pattern that cannot run under `re` should not get a verdict
from a tool whose correctness and ReDoS halves both run `re`.

Word boundaries (`\b`, `\B`) are supported. They look like lookaround and are
not: the condition depends only on the two characters either side of a
position, so a finite automaton can carry it in one bit of state. Deciding it
does require the alphabet to tell word characters from the rest, which is why
"every other character" is two symbols here rather than one.

In the dk.brics dialect this is a deliberate deviation from the spec, which
escapes `\b` to the literal character `b`. The corpora mean a boundary and
their paired descriptions say so — KB13 glosses `.*\b[A-Za-z]*er\b.*` as "lines
using words ending in 'er'", which the literal reading does not describe.

`\B` follows Python rather than mathematics: it is `¬\b` everywhere except the
empty string, which `re` refuses even though no boundary exists there. Patterns
scored here are run by `re`, so `re` is what gets modelled.

## Install

```bash
pip install regexbench
```

Python 3.10+. **No runtime dependencies** — stdlib only, deliberately, so this
drops into a training or CI pipeline without dragging anything with it.

## Status

Alpha: the API will change, and the version is 0.x for that reason. What is
stable is the discipline — every number in this README and in
[docs/benchmarks.md](docs/benchmarks.md) came from a run, and the equivalence
engine is differential-tested against Python's own `re` on every release.

Known limits, in the order they cost you coverage:

| Construct | Status |
| --- | --- |
| `^` / `$` away from the pattern ends, under `SEARCH` | `UNSUPPORTED` — the `.*p.*` rewrite has nowhere to put them. 9.8% of Re(gEx|DoS)Eval; resolved exactly under `FULLMATCH` |
| `$` inside a lookaround body, under `SEARCH` | `UNSUPPORTED` — the subject can always carry one more newline, which the fold cannot express. 1.2% of Re(gEx|DoS)Eval |
| Lookaround | Supported — `(?=…)`, `(?!…)`, `(?<=…)`, `(?<!…)` built into the automata. Refused for a variable-width lookbehind, a `\b` immediately in front of one or at the right edge of a lookbehind body, or one inside a dk.brics `&`/`~` operand. Nesting is decided only inside a positive lookahead, at the body's start, on every path |
| `$` before text that could be the subject's final newline, under `FULLMATCH` | `UNSUPPORTED` — Python's `$` matches there too, and folding the anchor cannot say so. 1.2% of Re(gEx|DoS)Eval; exact under `SEARCH` |
| Backreferences | `UNDECIDABLE` — no engine can answer this |
| `[\D0-9]` — a negated shorthand mixed with other members | `UNSUPPORTED` — not one character set |
| Possessive quantifiers, atomic groups | `UNSUPPORTED` unless the body matches exactly one way |

Correctness scoring and ReDoS screening have no such limit — they run the real
`re` engine and work on any pattern it compiles.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). The short version: a wrong answer is
worse than no answer, so anything the engine cannot decide has to say so.

Changes are listed in [CHANGELOG.md](CHANGELOG.md).

## License

Apache-2.0.

## Working in this repository

Project metadata and research context live in [.plicara/README.md](.plicara/README.md); agent constraints live in [AGENTS.md](AGENTS.md). Use `make setup` and `make check` for the default local environment and verification. Expensive experiments, model downloads, and publication are separate explicit steps. Project status is authoritative in `.plicara/project.yaml`; no central board update is required.
