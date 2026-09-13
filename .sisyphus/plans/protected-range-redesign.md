# Replace masking with a protected-range index

Handoff for the remaining performance work on obsidian-linter. Everything below was measured on
this machine against a real 866KB / 14,020 line document. Numbers are reproducible with the
harnesses given in "How to verify".

## Where to start

- Repo: `/home/saberzero1/Repos/obsidian-linter`, remote `fork` = `github.com/saberzero1/obsidian-linter`
- Branch: `perf/single-parse-masking`, tip `db84acc`
- Fixtures (untracked, do not commit): `Introduction.to.a.Self.Managed.Life.md`, `data-test.json`
- Run `npm install` first. There is a `patches/` entry that must stay applied, see "Traps" #1.

Rollback points if something needs unwinding:

| Commit | State | Lint time |
|---|---|---|
| `db84acc` | tip, adds rule batching | ~88s |
| `26c46d8` | before batching | ~90s |
| `27b0573` | before the restore fix | ~137s |
| `50a88c6` | before the seed fix | ~205s |
| `master` | untouched | ~420s |

## The problem, stated as a measurement

A lint of the 866KB file takes ~88s. Of that, ~33s is parsing markdown, spread over 36 parses of
the document. Those 36 break down as:

```
maskParses = 14    during ignoreListOfTypes, masking a rule's ignore types
bodyParses = 22    inside rule bodies, spread over 17 rules, mostly one each
```

The reason there is more than one parse at all: `Rule.apply` (`src/rules.ts:110`) wraps every rule
in `ignoreListOfTypes`, which **replaces protected regions of the document with placeholder
strings**. The rule body then runs on that rewritten text. Any mdast helper the body calls parses
that text. Every rule masks a different set of types, so every rule body sees a different document,
and no two can share a parse.

Rule batching (`db84acc`) already gives a run of rules the same input text, taking 43 rule steps
down to 12 batches. It did not reduce parse time, because it only addresses the 14 masking parses,
not the 22 inside bodies.

**The remaining win requires rules to stop receiving a privately rewritten copy of the document.**

## Goal

Rules consult a protected-range index over the *original* text instead of being handed a rewritten
copy, so that all rules in a batch share one parse.

### Success criteria

1. Linting the 866KB file with `data-test.json` produces **byte-identical output** to `db84acc`.
   If a difference is intended, it must be named and justified individually, not accepted in bulk.
2. `npx jest` is green. Currently 1261 tests across 63 suites.
3. `npx eslint src/ __tests__/` clean. `npx tsc --noEmit` reports 63 errors, the pre-existing
   baseline, and no more.
4. Parses during one lint of that file drop from 36 to **under 15**. This is the actual objective;
   without it the change is not worth its risk.
5. Lint time for that file under **60s**. Parsing is ~33s of the current 88s, so removing most of
   it should land near 55-60s. Do not expect more, see "What this will not achieve".
6. `__tests__/lint-is-idempotent.test.ts` still passes. Linting twice must not change a file.

## Design

Replace the "rewrite the document" strategy with "describe what must not be touched".

```
LintContext {
  text: string                 // the original, never rewritten
  ast: Root                    // one parse of it
  protectedRanges: Range[]     // merged, sorted, from one traversal
  isProtected(start, end): boolean
}
```

- Build the protected ranges from a single `visit` over the AST using an array test, plus the
  regex and function based ignore types. `unist-util-visit` accepts `visit(tree, ['code',
  'inlineCode', 'math'], fn)` so all mdast types come from one traversal.
- Merge overlapping ranges into a union. The outermost wins, which is what masking already does
  since `a4c87be`.
- Rules ask "is this range protected?" and skip it, instead of never seeing it.

This is how markdownlint works. It parses once per lint and rules query cached token ranges with an
overlap test rather than the source being rewritten. Reference implementations worth reading:
`markdownlint/lib/md011.mjs` uses `filterByTypesCached` plus `hasOverlap`; `remark-lint-maximum-
line-length` skips protected node types with `SKIP` during its own traversal.

### Staged migration

Do not convert all rules at once. The framework can support both contracts at the same time.

1. Build `LintContext` and the range index. Do not change any rule yet. Verify the ranges it
   produces cover exactly what masking currently hides, by asserting that for every ignore type and
   every document in the corpus, the set of masked spans equals the set of indexed ranges.
2. Convert the mdast helpers in `src/utils/mdast.ts` one at a time to take a context and skip
   protected ranges. There are 12 that rewrite by offset: `updateItalicsText`, `updateBoldText`,
   `updateListItemText`, `updateHeaderText`, `removeSpacesInLinkText`,
   `makeEmphasisOrBoldConsistent`, `makeSureThereIsOnlyOneBlankLineBeforeAndAfterParagraphs`,
   `updateOrderedListItemIndicators`, `updateUnorderedListItemIndicators`, `updateBlockquotes`,
   `ensureFencedCodeBlocksHasLanguage`, `ensureEmptyLinesAround*`. Each conversion is one commit,
   each gated by the differential.
3. Convert rules to the context contract, highest parse count first. From the measurement, the
   rules whose bodies parse are: `remove-space-before-or-after-characters` (4),
   `remove-space-around-characters` (3), then one each for `move-math-block-indicators-to-their-
   own-line`, `default-language-for-code-fences`, `emphasis-style`, `empty-line-around-math-blocks`,
   `move-footnotes-to-the-bottom`, `ordered-list-style`, `re-index-footnotes`, `remove-link-
   spacing`, `remove-multiple-spaces`, `space-between-chinese-japanese-or-korean-and-english-or-
   numbers`, `strong-style`, `unordered-list-style`, `yaml-title`, `blockquote-style`,
   `trailing-spaces`.
4. Keep `ignoreListOfTypes` for any rule not yet converted. Delete it only when nothing calls it.

Measure parses after every step. If a step does not reduce them, find out why before continuing.

## Traps

Each of these cost real debugging time. They are in rough order of how much damage they do.

1. **`patches/mdast-util-from-markdown+2.0.3.patch` must stay applied.** `prepareList` splices two
   events into the shared event array per list item, which is quadratic. On this file that was 80%
   of parse time. `postinstall` runs `patch-package`. Delete the patch only when a release contains
   the fix for `syntax-tree/mdast-util-from-markdown#49`. If parse times suddenly triple, check
   this first.

2. **Non-overlapping edits are not safe to merge.** Two rules can change different characters and
   still produce nonsense together, because markdown constructs span lines. Merging a list marker
   rewrite with a blank line rewrite produced `- [- [` in the document. `src/utils/text-edits.ts`
   widens each edit to its surrounding lines before testing for clashes. Do not narrow that back to
   exact ranges.

3. **Nested nodes of one type report overlapping positions.** `*)*g**` yields emphasis at `[2,5]`
   and `[0,6]`. Replacing the inner one shifts the outer one's offsets, and the second replacement
   cuts into the first placeholder, duplicating text and leaving an unrestorable placeholder. Fixed
   by collapsing overlaps to the outermost, `src/utils/ignore-types.ts`. The range index must do the
   same. Nested blockquotes and nested lists hit this too.

4. **Ignore type order is load bearing.** `anchorTag` must be handled before `html`, because an
   anchor is an opening and a closing html node that do not cover the url between them; get it
   wrong and `no-bare-urls` wraps the url inside the anchor. Also `yaml` before `thematicBreak`
   (frontmatter delimiters are a valid thematic break), and `anchorTag`/`link`/`wikiLink` before
   `url`. The canonical order and its reasoning are in `src/utils/ignore-types.ts`.

5. **Two rules read placeholder text directly.** `src/rules/blockquote-style.ts:66-67` builds
   regexes from `IgnoreTypes.math.placeholder` and `IgnoreTypes.code.placeholder`.
   `src/rules/space-between-chinese-japanese-or-korean-and-english-or-numbers.ts:33` does the same
   for link, inline math, inline code and wiki link. Both must be converted before placeholders can
   be removed, or they will silently stop matching.

6. **Yaml rules build on each other.** `insert-yaml-attributes` adds `tags: ` and an array rule
   then decides it should be `tags: []`. Giving them a shared snapshot broke 145 of 183 documents.
   They are barriers in `src/rules-runner.ts` and must stay that way.

7. **Some rules depend on earlier rules having run.** `line-break-at-document-end` adds a trailing
   newline that `move-footnotes-to-the-bottom` then strips. No textual clash test catches this,
   because one rule changes whether the other needs to act. The list is
   `rulesThatMustSeeEarlierWork` in `src/rules-runner.ts`. Expect to add to it.

8. **`getPositions` returns a copy on purpose.** Callers mutate it, `removeOverlappingPositions`
   pops from it. Returning the cached array corrupts the cache. See the comment in
   `src/utils/mdast.ts`.

9. **Placeholder length changes output.** Shortening the suffix from 16 to 13 characters changed
   the linted document. If placeholders survive anywhere, keep them 16 characters.

10. **Restore is case-insensitive deliberately.** Rules such as `capitalize-headings` change the
    case of placeholder text, see issue #201. Any replacement mechanism needs the same tolerance.

11. **The parse cache is keyed on a 53 bit hash plus an exact text comparison.** The comparison is
    not redundant; without it a hash collision hands back another document's AST.

## How to verify

Three gates. Run all of them on every commit.

### 1. The test suite

```bash
npx jest && npx eslint src/ __tests__/ && npx tsc --noEmit -p tsconfig.json
```
1261 tests, eslint clean, exactly 63 tsc errors.

### 2. The runner differential

This is what caught every regression in this work. Dump the lint output of a corpus, stash, dump
again, compare. Write `__tests__/zz-runner-diff.test.ts` that lints every document in a corpus and
writes `index\u0001JSON.stringify(output)` per line to `process.env.DUMP_PATH`. Corpus = every
rule's `examples[].before`, plus the first 600 lines of the 866KB file, plus documents built to
exercise the traps above. Then:

```bash
DUMP_PATH=/tmp/new.txt npx jest __tests__/zz-runner-diff.test.ts
git stash push -- src/
DUMP_PATH=/tmp/old.txt npx jest __tests__/zz-runner-diff.test.ts
git stash pop
diff /tmp/old.txt /tmp/new.txt
```

**Check the stash actually reverted something.** A comparison against code that is already
committed compares identical code and passes meaninglessly. This happened during this work. If the
change is committed, compare against `<commit>~1` with `git checkout <commit>~1 -- src/...`.

Adversarial documents worth including, each targeting a trap: an anchor tag wrapping a bare url,
frontmatter followed by `---`, `<% %>` containing markdown, `*)*g**`, nested blockquotes with empty
lines, nested lists, a checklist next to a blank line, a heading ending in `!!!`, footnotes, a
document that is only whitespace, and the empty string.

### 3. Parse count and time

Mock `mdast-util-from-markdown` and count calls to `fromMarkdown` while linting the 866KB file:

```ts
const mockStats = {calls: 0, ms: 0};
jest.mock('mdast-util-from-markdown', () => {
  const actual = jest.requireActual('mdast-util-from-markdown');
  return {...actual, fromMarkdown: (text, ...args) => {
    const start = performance.now();
    const result = actual.fromMarkdown(text, ...args);
    mockStats.calls++; mockStats.ms += performance.now() - start;
    return result;
  }};
});
```

The variable must be named `mock*` or jest rejects the factory. Note lint timings vary by about
10% run to run, so do not trust a single measurement of a small change; parse **count** is stable
and is the better signal.

## What the first conversion measured

Read this before judging a step by its lint time.

Working out a document's protected ranges costs about as much as parsing it. On the 866KB file the
ranges for one rule's ignore types take ~1.2s and a further ~0.9s for the `list` and `html` a rule
masks inside its own body, against ~1s for a parse. Nearly all of that is the mdast traversals and
the `tag` and `wikiLink` expressions over the whole document; the rule's own expressions are 4ms.

That cost is per document, not per rule, and is cached, so it is paid once and shared by every
converted rule that sees the same text. Masking paid it per rule but on a *smaller* document, since
each stage shrinks the text the next one scans.

The consequence is that converting one rule is net negative on wall time and only the parse count
improves. The lint time turns around once enough rules share the ranges to cover their cost, which
is what happened by the third rule:

| After converting | Parses | Parsing | Lint |
|---|---|---|---|
| nothing | 36 | 33.5s | 87.0s |
| `remove-space-before-or-after-characters` | 33 | 32.4s | 94.3s |
| `remove-multiple-spaces` | 31 | - | - |
| `remove-space-around-characters` | 29 | 27.6s | 83.9s |
| `emphasis-style`, `strong-style` | 28 | 28.7s | ~85s |
| `ordered-list-style`, `unordered-list-style` | 26 | - | - |
| `default-language-for-code-fences`, `remove-link-spacing` | 25 | - | - |
| `trailing-spaces` | 25 | 25.9s | 83.8s |
| the two footnote rules | 23 | 22.9s | 81.9s |
| `blockquote-style`, `space-between-chinese-...` | 21 | 21.2s | 80.9s |
| eight more expression rules | 21 | 21.6s | 76.4s |
| the rest, and masking deleted | 17 | 17.9s | 73.1s |
| one tree walk per parse instead of six | 17 | 17.0s | 58.8s |
| not rehashing the document per cache lookup | 17 | 16.1s | 31.3s |
| keying the caches on the document itself | 17 | 17.1s | 26.8s |
| taking micromark's own quadratic fixes | 17 | 10.4s | 20.1s |
| batching the bullet and link edits | 17 | 9.4s | 16.4s |
| batching the emphasis and strong delimiters | 17 | 9.4s | 14.6s |
| normalising paragraph spacing by gaps | 17 | 9.2s | 13.6s |
| sharing a snapshot across the four cleanup rules | **15** | 8.3s | **12.6s** |

Twenty-two rules converted. All 226 corpus documents byte identical against the source as it was
before any of this work, throughout.

### The one thing standing between here and the goal

Six rules are left on masking for the same reason, and they are the reason `ignoreListOfTypes`
cannot go yet:

- `move-math-block-indicators-to-their-own-line`
- the four `empty-line-around-*` rules
- `remove-empty-list-markers`
- `remove-trailing-punctuation-in-heading`
- `heading-blank-lines`

Every one of them decides something from **where a line begins or ends**, and masking replaced a
multi line ignored construct with a **single line** token, so the document they were reasoning about
had a different line structure from the real one. `redactProtected` solves the half of this where a
rule reads neighbouring syntax, but not this half: the window itself is the wrong shape.

### How to solve it: project the document, do not mask it

The framing that kept this stuck was treating "reproduce masking's line structure" as the same thing
as "reproduce masking". It is not. **Masking's cost was never the string rewriting. It was that the
rewritten text was handed to the rule, so every mdast helper the rule called parsed that text
instead of the shared one.** A copy of the document used *only to decide things*, with node
positions still taken from the shared parse of the original, costs no parse at all.

So build, per document and per ignore type set, and cache on the `LintContext` next to the ranges:

- a **projection**, the original text with each protected range replaced by a single line token.

  **Byte identity with the masked text is not reachable in one pass, and the argument below that
  claimed otherwise was wrong.** `getSeedForText` seeds from the length of the text *the current
  stage sees*, not the original, and then probes that candidate against that stage's whole text,
  which already contains earlier placeholders. So the same document yields different seeds at
  different stages: for "wiki links and tags next to urls" under `no-bare-urls` the link, wiki link
  and tag stages seed from lengths 90, 101 and 127 and produce three different suffixes, where a
  single pass over the original would use one. Intermediate lengths alone do not fix it either,
  because the collision probe can push the seed on, and earlier replacements can create or destroy
  the candidate string. `mergeRanges` also throws away which type each range came from, while
  masking emits a type specific placeholder per range.

  Reproducing the bytes therefore needs a stage aware replay, and the mdast stage of that replay
  needs the staged text parsed, which is the cost being removed. So that route is closed.

  What to do instead: give each range a token that is **structurally** what masking's placeholder
  was, a single line, non whitespace, not resembling markdown syntax, and **the same length** as the
  placeholder that type would have produced. Trap #9 records that placeholder length has changed
  the linted document before, so length is not a free choice. Correctness then rests on the corpus
  differential per converted rule, which is the standard every other conversion here has been held
  to, rather than on an equality proof.

  Before relying on it, check the weaker property that is still provable: that the projection has
  the same line count as the masked text, and that the two agree line for line on which lines are
  blank. That is what the six blocked rules actually read, and it is cheap to assert over the
  corpus;
- an **offset map** both ways between projection coordinates and source coordinates. The ranges are
  sorted and disjoint, so this is a pair of parallel arrays and a binary search, the same shape as
  `ProtectedRanges` itself.

A rule then:

1. runs its line oriented expressions against the **projection**;
2. maps each resulting edit range back to **source** coordinates;
3. drops any edit that lands inside a placeholder, since that is a protected range;
4. takes any mdast node positions it needs from the **original** text as it does now, mapping them
   into projection coordinates through the map where the two have to meet.

Step 1 is what makes the six tractable: `heading-blank-lines` counting blank lines, the
`empty-line-around-*` family looking at the line either side of a construct,
`remove-empty-list-markers` taking a blockquote prefix with its match, and
`move-math-block-indicators` reading the line a block starts on, all see exactly the document they
saw before.

`redactProtected` becomes the window sized special case of this and should be reimplemented on top
of it rather than kept separate.

Cost: one string build of the document per ignore type set per batch, no parse. Masking paid that
too, and then paid for a parse on top. Cache it on the context so the rules in a batch that share an
ignore set share the projection, the way they already share the ranges. Watch the parse count and
the lint time after the first rule moves onto it; if the projection is being rebuilt per rule rather
than per batch it will show up immediately, which is exactly what happened when a rule built its own
`LintContext`.

What to verify first, before converting anything onto it: assert on the corpus that the projection
and the text `ignoreListOfTypes` produces have the same number of lines and agree on which of those
lines are blank. Full equality is not available, for the reasons above, but that weaker property is
exactly what the six blocked rules read, and it is cheap to check over the corpus for every ignore
type set any rule declares. If it holds, the six conversions become ordinary work gated by the
differential.

Until this exists those six hold masking alive, and with it the ~14 parses that `ignoreListOfTypes`
costs.

The remaining parses split roughly as: one per batch for the ranges, which is the floor, plus the
masking parses those six rules still force.

The parse counts in this table are for the whole 866KB document. The cheap 600 line excerpt the
default test run measures went 30 to 20 over the same commits, so it moves faster; use the full
fixture when comparing against the goal.

**So: judge a step by the parse count, which is the thing being removed, and only expect lint time
to fall once most of the rules in a batch are converted.** Do not revert a step that reduced parses
because it cost a few seconds.

Two things follow for the order of the remaining work:

- Convert rules that share ignore types together, so the ranges they share are worked out once.
  `remove-space-around-characters` and `remove-multiple-spaces` have almost the same list as the
  rule already converted and do the same nested `list` masking, so they are the next slice.
- The nested masking a rule does inside its body needs `ProtectedRanges.combinedWith`, which goes
  back through the context so the combination is cached. Building a `LintContext` inside a rule
  instead costs ~0.9s on every call and was worth ~5s of the first measurement.

## How to convert the rest

Five rules are converted: `remove-space-before-or-after-characters`, `remove-multiple-spaces`,
`remove-space-around-characters`, `emphasis-style`, `strong-style`. Read one of them before
starting; they are the worked examples.

### The one question that decides every conversion

Every change has two ranges, and they are not the same:

- the **edit range**, the characters actually rewritten;
- the **guard range**, the characters masking would have had to *show the rule* for the change to
  happen at all. Skip the change when the guard range is protected.

The guard range differs per rule, and the test is always: **would the placeholder that stood in for
the ignored region have satisfied what this rule matches on?**

| The rule anchors on | A placeholder is | Guard on | Example |
|---|---|---|---|
| any non whitespace | one, so masking did make the change | the edit range only | `remove-multiple-spaces` |
| specific characters the user configured | not one | the whole match | `remove-space-*-characters` |
| an mdast node's delimiters | irrelevant, the interior is copied through | the delimiters only | `emphasis-style` |

Getting this backwards is the main way to break a conversion, and three of the five needed it
corrected. The enclosing versus enclosed case is the one that catches people: formatting *around*
an ignored region is editable, because masking replaced only the region and left the delimiters
visible; formatting *inside* one is not.

### Mechanics

- `usesProtectedRanges: true` in the rule's `super({...})`, third parameter
  `protectedRanges: ProtectedRanges`. Leave `ruleIgnoreTypes` alone.
- For the second set of types a rule masks inside its own body, use
  `protectedRanges.combinedWith([...])`. **Never** construct a `LintContext` inside a rule; it
  costs ~0.9s per call on the large fixture because nothing is cached.
- Use `collectUnprotectedRegexReplacements` in `src/utils/protected-ranges.ts` for regex driven
  rules. It takes the edit range and the guard range separately.
- Collect every change against the text the rule was given and apply them once with
  `replaceTextRanges`. Do not rewrite the text between passes.
- A helper with a single caller moves with that caller, in place. A helper with several callers
  gets a second version beside it until the last caller has moved.

### Verification, every step

```bash
npx jest && npx eslint src/ __tests__/ && npx tsc --noEmit -p tsconfig.json   # 63 errors exactly
DUMP_PATH=/tmp/new.txt npx jest __tests__/zz-runner-diff.test.ts
diff /tmp/committed.txt /tmp/new.txt                                          # must be empty
npx jest __tests__/zz-parse-count.test.ts                                     # must not go up
```

Dump `/tmp/committed.txt` from the last commit before starting, and re-dump it after every commit;
a stale baseline is worse than none. To check against the linter as it was before any of this work,
`git checkout 37d1f70 -- src/`, dump, then `git checkout HEAD -- src/`.

The corpus missed a real difference once, because it had no document with two spaces next to a
link. When a conversion turns on a boundary the corpus does not contain, add the document.

### Expect conflicts, and do not guess

Four of the five conversions turned up an input where the index and masking genuinely disagree.
Two were worth accepting as named differences, one was a pre-existing corruption bug, and one
stopped the work. When one appears, write down the smallest input that shows it, and decide
whether masking's answer was intended behaviour or an artifact of what the placeholder looked like.
Artifacts are not worth reproducing; intended behaviour is.

### A rule has to be told what it may see, not only what it may write

Everything above is about which characters a rule may **change**. There is a second kind, found
converting the `empty-line-around-*` family: a rule that **reads** the text around its target to
decide what to do.

`empty-line-around-math-blocks` ignores `code`. Given this document:

````markdown
> ```
> code
> ```
> $$
> x
> $$
````

masking produces a blank quote line between the two:

````markdown
> ```
> code
> ```
>
> $$
> x
> $$
````

and computing from the original text leaves the document alone. `codeBlockBlockquoteRegex` finds
the neighbouring fence in the original text and takes the branch that preserves the spacing;
masking had replaced that fence with a placeholder, so the expression did not match and the other
branch ran. Swapping the two blocks shows the same thing on the other side. No write guard can fix
it, because the difference happens before any edit is chosen.

`redactProtected` in `src/utils/protected-ranges.ts` is for this: it returns a short window of the
document with the protected ranges in it replaced by a neutral token, which is what the expression
would have seen under masking. It needs no parse and is bounded by the window, so it does not cost
the win back. Its result is for **decisions only**; the offsets in it are not source offsets and
must never be used to locate an edit.

### The hard class is rules that reason about lines

A pattern has emerged across the conversions. The rules that convert cleanly are the ones that work
on **spans**: a regex match, an mdast node's delimiters, an insertion point. The ones that fight
back are the ones that reason about **which line something is on**, and they fight back for the
same reason every time: masking replaced a multi line construct with a **single line** token, so the
text around it became adjacent in a way it is not in the original, and the rule's idea of where a
line starts and ends moved with it.

Two are deferred for this reason.

**`move-math-block-indicators-to-their-own-line`.** For

````markdown
> `x
y` $$x$$
````

masking gives

````markdown
> `x
y`
> $$
> x
> $$
````

and the original text gives the same thing without the `>` prefixes on the math. The inline code
span runs across two lines, masking collapsed it to one, and that left the `>` visible on the line
the math effectively starts on. Redacting the physical line window does not recover it, because the
window itself is the wrong shape. A conversion attempt also turned up a second, unrelated problem:
`breakMathBlockIntoMultipleBlocksIfNeedBe` can return overlapping ranges, which a single pass
collection of replacements does not survive, and it inserted a stray `$$$$` on malformed input. None
of the 226 corpus documents change either way, but the stray `$$$$` is a corruption and not
something to accept.

### `yaml-title` costs a parse rather than saving one

A conversion of `yaml-title` was written and dropped. It behaved correctly, the corpus did not
change, and it still **raised** the parse count from 20 to 21 on the excerpt. The rule runs from
`runAfterRegularRules`, on its own rather than in a batch, so the ranges it asks for are worked out
for a document nothing else is looking at, and that costs more than the parse it saves. Check the
parse count before assuming a conversion is worth having; a rule outside the batched run has
nothing to share with.

It also turned up a third pre-existing corruption, not yet decided. For

````markdown
#
```
```
````

the heading expression's `\s+` runs across the newline and swallows the masked code block, so the
title ends up holding the whole block and the frontmatter comes out as a `title: |-` block scalar
with no indentation, which is not valid yaml. Whoever converts this rule has to decide whether to
reproduce that or to leave the heading alone when the match crosses into a protected region.

### The `empty-line-around-*` family is deferred, on purpose

Only `empty-line-around-math-blocks` of the four contributes a parse from its body; the other three
do not appear in the measurement at all. Against that,
`makeSureContentHasEmptyLinesAddedBeforeAndAfter` reaches through
`makeSureContentHasASingleEmptyLine*ForBlockquote`, `getIndexOfEndOfLastNonEmptyLine` and
`getIndexOfStartOfFirstNonEmptyLine`, all of which interleave reads of the surrounding text with
the offsets they then write at. Redaction cannot simply be threaded through that, because the same
string is used for both.

So convert the cheaper rules first and come back to this family last, when it is the only thing
keeping `ignoreListOfTypes` alive and the cost is worth paying.

### The last rule, and why it is last

`paragraph-blank-lines` is the only rule still on masking, and it is what keeps `ignoreListOfTypes`
alive. Two attempts have been made and both reverted.

The first found that `getProjectedNodeRanges` drops a node whose **start** maps inside a token, but
a paragraph can begin inside a protected table and run past it, so the visible part loses its edits:

```markdown
# H
| a |
| - |
| b |
Paragraph
Next
```

masking puts a blank line either side of the table and around each paragraph; mapping node ranges
dropped the paragraph whole and changed nothing.

The second clamped such a node to its visible extent, which fixes that case, and then hit a second
one:

```markdown
A
%%
B
%%
```

masking puts a blank line after `A` and the converted rule does not. The reason is that
`obsidianMultiLineComments` is a **regex** ignore type, not an mdast node, so mdast reads the whole
of that document as **one paragraph**. Masking replaced the comment with a token and the rule then
saw a paragraph followed by a one line token; on the original text there is a single paragraph node
spanning everything, and clamping it to its visible extent is not the same thing.

So the general shape of the problem is: for this rule the **node boundaries themselves** differ
between the two worlds, not merely their offsets. Anyone finishing it should work out whether the
paragraph positions should come from parsing the projection rather than the original. That would
cost one parse of a different text and needs weighing against what removing masking saves, which is
the remaining ~14 parses; it may well be worth it, and it is the one place in this work where
parsing something other than the original could be the right answer.

Both reverts are clean; the tree at that commit is green and byte identical.

### Deleting the masking costs a parse, so it is a trade rather than a tidy up

Every rule that declares `ruleIgnoreTypes` is converted. Three callers of `ignoreListOfTypes`
remain: `yaml-title` and `yaml-title-alias`, which mask **inside their bodies** to find the first
level one heading, and the fallback branch in `Rule.apply` that exists for them.

Those two were converted, together, on the theory that they would then share one context and one set
of ranges for the same text. **They do not, and the parse count went from 12 to 13.** They run from
`runAfterRegularRules`, outside the batched run, and each of them changes the text, so neither the
context nor the ranges are shared with anything. Working the ranges out for a text nothing else
looks at costs about what parsing it costs, which is the same thing that made converting `yaml-title`
alone a loss earlier. Converting both at once does not rescue it.

The conversion itself was otherwise good: it selects the first heading whose whole match is
unprotected, and it declines to reproduce the invalid yaml the old code produced for a heading whose
text runs into a code block, with **0 of 226 corpus documents changing**. The work in progress is
kept at `.sisyphus/plans/yaml-title-wip.patch` against `0899034`.

So the decision is a trade, and it is not obviously worth taking:

- **Convert them**, delete `ignoreListOfTypes`, the `Rule.apply` branch and the dead helpers, and
  end with a single mechanism and no masking anywhere, at a cost of **one parse**.
- **Leave them**, keep the parse, and keep the masking machinery alive for two rules that do not
  benefit from moving.

Since the goal is fewer than 15 parses and the 866KB lint currently sits at 16, converting them
moves away from the number this work exists to improve. That is why it has not been done.

Worth noting for whoever decides: the remaining parses are no longer masking. They are roughly one
per batch for the context, which is the floor of this design. Getting under 15 is now a question
about how many batches the runner makes, not about masking.

### What the time actually went on, once it was measured

The plan assumed parsing was the cost and that removing masking would remove it. Parsing was 33.5s
of 87.0s, and removing masking took it to 17s. That left ~50s nobody had accounted for, and two
guesses about it were wrong before anything was measured:

| Suspected | Actual |
|---|---|
| `getEditsBetween` diffing the whole document per rule, no timeout | **237ms**. diff-match-patch trims the common prefix and suffix first, and these strings differ by a handful of edits |
| the clash test widening every edit to its surrounding lines | **1ms** |
| applying the edits | **26ms** |
| the regex scans for `tag` and `wikiLink`, which use unicode property escapes | **84ms** across 47,487 calls |
| working out the protected ranges | **45,682ms of 68,711ms**, two thirds of the lint |

All of that 45.7s was walking the tree: 26.3s for the ignore types a rule declares, 19.3s for the
types the mdast helpers ask for directly. The second number is the one that had been missed.
Paragraphs, list items and footnote definitions are never ignore types, so they were never part of
the set collected in one walk, and each request for them walked the whole tree again. About six
walks per parse.

Collecting every type on the first walk took the lint from 73.1s to 58.8s, measured back to back.

**The lesson, which cost two wrong guesses to learn: measure the breakdown before optimising it.**
The diffing was the obvious suspect, it was architecturally interesting to remove, the refactor had
already put the information in place to remove it, and it was worth 0.3% of the run.

### The tree walk was never the cost: it was hashing the document to look things up

After the walk was reduced to one per parse, `getPositions` still measured 36.5s over 76 calls, so
the walk looked like what was left. It is not. On this document:

| | |
|---|---:|
| nodes in the tree | 28,307 |
| a plain recursion collecting every position | **3ms** |
| `unist-util-visit` with no test | 37ms |
| `unist-util-visit` with a list of types as its test | 94ms |
| sorting every bucket afterwards | 2ms |
| parsing | 751ms |
| **`hashString53Bit` over the 866KB document** | **288ms** |
| **the same, 76 times, which is what a lint did** | **21.6s** |

Both the parsed markdown and the protected ranges are kept in caches keyed on a hash of the
document, and `getPositions` goes through the parse cache on every call. So each call read all
866KB to compute a key it had already computed. Remembering the last document's hash took the lint
from 59.1s to 31.3s, and hashing then still cost 5.3s for the seventeen documents it had to do.

Keying the caches on the **document itself** removed the rest. The engine keeps a string's hash on
the string once it has been used as a key, so it is computed once per document and never again, and
Trap #11 goes away with it: there is no bucket to collide in, so nothing has to compare the text
afterwards to check it got the right one. 31.3s to 26.8s, and `hashString53Bit` now has no callers.

Two things worth keeping from how this was found. The instrumentation attributed the time to
`getPositions`, which was true and misleading: the cost was inside the cache lookup it makes first,
not the walk it is named for. And the walk had already been optimised twice on the assumption it
was expensive, when it was three milliseconds all along. **Attribution by wrapper tells you which
function, not which line.**

### Where the remaining time is, measured after the tree walk fix

| | ms | calls |
|---|---:|---:|
| `getPositions`, which does the one walk per parse | 36,461 | 76 |
| `protectedRangesFor`, which is almost entirely the above | 36,515 | 81 |
| `projectionFor` | 23 | 19 |
| `getAllTablesInText` | 8 | 2 |
| `getAllCustomIgnoreSectionsInText` | 0 | 0 |

So **the walk itself is now about 62% of a 59s lint**, at roughly 2.1s each across 17 parses, and
everything else in the range machinery is noise. The two leads left over from the earlier map are
both dead: the custom ignore scan does not run at all on this document, and the table scan is 8ms.
Sharing range unions across snapshots would save nothing either, since the unions are not where the
time goes.

One thing was tried and reverted: handing `visit` a list of sixteen types makes it test every node
against the whole list, so the walk was changed to visit every node and do one map lookup instead.
That is 38.6s to 36.5s measured on `getPositions` directly, but it does not survive into the lint
time, which stays at 59s either way. Not worth a micro optimisation carrying a comment that claims
a benefit the end to end number cannot show.

If anyone wants the next chunk, it is the walk. `unist-util-visit` maintains ancestor chains and
handles its skip and exit protocol on every node, none of which this needs; a hand written recursive
walk that pushes positions into a map is the obvious thing to try, and it should be measured on
`getPositions` **and** on the lint before being kept.

### Parsing is now everything, and it does not scale linearly

At 26.8s the profile is clean: `fromMarkdown` is 17.1s, the tree walk is 0.6s, diffing 0.2s, edits
24ms, the clash test 1ms. Every remaining avenue is the parse.

Measured on slices of the 866KB document, best of three runs with the **largest measured first** so
that neither warmup nor garbage collection favours it:

| size | parse | per KB |
|---|---:|---:|
| 262KB | 150ms | 570µs |
| 487KB | 412ms | 845µs |
| 625KB | 677ms | 1084µs |
| 840KB | 1003ms | 1195µs |

Per-KB cost rises monotonically: roughly O(n^1.6). **If parsing were linear at the rate this
document shows at 262KB, the full parse would be about 480ms rather than 1000ms**, so something
between 8 and 9 seconds of the 26.8s is a scaling problem rather than work that has to happen.

This parser already carries one quadratic fix, `patches/mdast-util-from-markdown+2.0.3.patch` for
`prepareList`, which was 80% of parse time when it was found. It is reasonable to expect another.

Two further measurements narrow it:

- **Tokenising is about 70% of a parse and building the mdast tree about 30%**: `micromark` alone on
  the 840KB document is ~900ms against ~1200ms for `fromMarkdown`. So replacing the tree with the
  raw token stream, which several rules could probably live with, is worth at most a third.
- Bisecting by extension was **too noisy in one process to localise anything**; the same size varied
  by 40% between variants. What it did show is that the superlinearity is present with **no
  extensions at all**, so it is in micromark itself rather than in frontmatter, footnotes, task
  lists or math.

**It was a second `prepareList`, and it was already fixed upstream.** The lock held
`micromark@4.0.0` and `micromark-util-subtokenize@2.0.0`, both predating two fixes for quadratic
behaviour in micromark's event array: `micromark/micromark#171`, which took the reporter's 500KB
samples from 55 seconds to under 2 and shipped in `micromark-util-subtokenize@2.0.1`, and
`micromark/micromark#185`, which took a repeated label pattern from 500ms to 3ms and shipped in
`micromark@4.0.1`. Updating both took parsing from 17.1s to 10.4s and the lint from 26.8s to 20.1s.

Two things worth keeping from that. The superlinearity was measured locally and correctly, and the
cause was a **dependency version**, which no amount of profiling this repository's own code would
have found. And `npm update` on two transitive packages was worth more than every change made to
this codebase's own parsing since the masking was removed.

The patch for `mdast-util-from-markdown` is still needed: its own list quadratic, issue #49, is
fixed on `main` in commit `53875a7` but has not been released beyond 2.0.3. Delete the patch when a
release containing that commit exists, not before.

### The profile at 20 seconds

Parse scaling was re-measured after the update rather than assumed, best of three with the largest
first: 635µs/KB at 262KB, 659 at 487KB, 722 at 625KB, 721 at 840KB. **Flat.** The climb to 1195µs/KB
is gone, and the 1.14x drift that remains looks like allocation rather than an algorithm.

Where the 20.1s goes:

| | ms | calls |
|---|---:|---:|
| parsing, inside `getPositions` | 10,400 | 17 |
| `getPositions` total, so ~0.9s of its own work | 11,306 | 76 |
| `protectedRangesFor`, which is almost entirely the above | 11,353 | 81 |
| `getEditsBetween` | 227 | 32 |
| `regex edit collection` | 91 | 47,487 |
| `projectionFor` | 23 | 19 |
| `replaceTextRanges` | 21 | 45 |
| `clash test` | 1 | 19 |

That accounts for about 11.7s. The remaining **~8.5s is the rule bodies themselves**, 45 rules each
making at least one pass over 866KB, which is exactly what "What this will not achieve" predicted
would be the floor of this design.

So the shape now is roughly **half parsing, two fifths rules, one twentieth machinery**, and the
machinery is done. Going further means one of:

- **fewer parses**, which is the batch structure question: worth about 3 of the 17, and every route
  to them changes rule ordering;
- **a faster parser.** The one published benchmark that includes micromark puts `markdown-it` about
  24x faster and `commonmark.js` about 30x on a small fixture, so there may be a lot here, but it is
  a parser migration and every helper is written against mdast;
- **fewer rule passes**, which is a different design again.

### Everything left scales with the number of snapshots, not the number of rules

Timing every rule individually gives a **flat** distribution, not a few expensive ones:

```
unordered-list-style       1961ms      space-after-list-markers    749ms
paragraph-blank-lines      1592ms      heading-blank-lines         748ms
strong-style               1495ms      quote-style                 743ms
ordered-list-style         1028ms      move-footnotes-to-the-bottom 727ms
remove-link-spacing         989ms      file-name-heading           721ms
trailing-spaces             862ms      convert-bullet-list-markers 717ms
```

`convert-bullet-list-markers` swaps one bullet character for another and costs 717ms. There is a
floor of roughly 700ms that every rule pays regardless of what it does, and the total across rules
is 20.6s, which is the whole lint. That is the clue: **`Rule.apply` wraps the parse and the range
computation**, so the earlier split into "10.4s parsing and 8.5s rule bodies" was mis-attributed.
Most of what looked like rule work is shared infrastructure charged to whichever rule happened to
trigger it.

What actually drives the cost is that **each new snapshot needs a fresh parse and a fresh set of
range scans**. The caches are per `LintContext`, and a `LintContext` belongs to one document, so
every one of the 17 snapshots pays for: one parse, plus one scan per ignore type it uses. Thirty
nine rules declare ignore types between them but only about 25 distinct **sets**, over about 15
distinct types, and the most common set is shared by 11 rules. Within a snapshot that sharing works.
Across snapshots nothing is shared at all.

So the remaining ~19s is roughly `17 x (parse + range scans)`, and the rule bodies proper are a
small part of it. **This re-values the batch structure work.** It is not worth "about 3 parses",
it is worth about 3 snapshots out of 17 of everything above, which is nearer 3.5s than 1.8s.

It also suggests a cheaper question first: **are the range scans shareable across snapshots?** A
rule's edits are known, so tags, wiki links and urls outside an edited region cannot have moved. The
same argument that makes shifting positions attractive applies to shifting ranges, and ranges are a
much simpler structure than a tree.

### The dependency avenue is now exhausted, with one question open

Every runtime dependency was audited against its published releases for performance fixes landing
after the pinned version. After taking the micromark ones, **there is nothing else to collect**:

- `yaml` is at 2.9.1 and already has the cached anchor and alias resolution from PR #612, which
  mattered only for documents with many anchors anyway;
- `unist-util-visit`, `unist-util-is`, `diff-match-patch`, `quick-lru`, `moment` and every micromark
  and mdast extension in use have no published performance fix after their pinned version;
- `micromark` and `micromark-util-subtokenize` are now current, and subtokenize 2.0.3 additionally
  fixed a stack overflow on large spreads that would have been waiting for us.

The one open question is the local patch. Upstream's own fix for the `prepareList` quadratic, PR #51
and commit `53875a7`, is **still unpublished**, so the patch has to stay. But upstream's version is
not the same as ours: it adds a small list fast path, a threshold above which it stops using spread,
and an in place suffix shift, and its PR reports 36 to 41 percent on wide lists and 11 to 15 percent
on a 564KB document. **Ours may be the slower of the two.** Porting their implementation into the
patch and measuring is a contained experiment worth doing before anything larger.

### Shifting ranges instead of rescanning: assessed, not attempted

The idea was to carry a snapshot's protected ranges into the next one by moving them past the edits
that produced it, rather than rescanning the document. The edits are known exactly and the place to
do it is `src/rules-runner.ts`, at `text = replaceTextRanges(snapshot, batchedEdits)`, where both
snapshots and the sorted edit list are in scope. Neither `runBeforeRegularRules` nor
`runAfterRegularRules` has an edit list at all, so they would need one derived first.

The blocker is which types can be shifted:

- **Thirteen of the twenty four come from `getPositions`** and therefore from the tree: `code`,
  `inlineCode`, `image`, `thematicBreak`, `italics`, `bold`, `list`, `blockquote`, `math`,
  `inlineMath`, `html`, `heading`, `link`. Shifting these needs a proof that the edit did not change
  the parse, which is incremental parsing by another name.
- **Eight come from a regex** and could in principle be shifted: `yaml`, `wikiLink`,
  `obsidianMultiLineComments`, `footnoteAtStartOfLine`, `footnoteAfterATask`, `url`, `anchorTag`,
  `templaterCommand`. Several are still boundary sensitive: `url` is greedy so an adjacent character
  changes its extent, `yaml` matches only the first block and its end depends on following text.
- **Three come from a finder**: `tag` excludes the whitespace in front of it, so the character
  before it matters; `table` reads following rows and preceding lines; `customIgnore` pairs a start
  marker with a later end marker.

And the rules are not gentle: they insert and delete newlines, rewrite whole yaml sections, and move
footnotes to the end of the document.

**The arithmetic does not favour it, and this was measured rather than reasoned about.** Timing one
cold scan of the whole 866KB document per detector:

| | ms | | ms |
|---|---:|---|---:|
| `url` | 6 | `tag` | 0 |
| `table` | 2 | `wikiLink` | 0 |
| `yaml` | 0 | `templaterCommand` | 0 |
| `anchorTag` | 0 | `obsidianMultiLineComments` | 0 |
| `customIgnore` | 0 | the two footnote detectors | 0 |

**About 8ms for all eleven**, and across a whole lint `protectedRangesFor` costs ~48ms more than the
`getPositions` inside it. The regex scanning is not worth optimising, so shifting ranges would buy
nothing and **should not be built**.

> **This closure was later found to be half wrong, and the error is instructive.** The 8ms figure is
> the cost of re-running the *regex detectors*. It correctly rules out shifting the eight regex
> types and the three finder types. It says nothing about the **thirteen tree-derived types**, whose
> cost is not a scan but a **550ms parse** — the note above even says shifting those "needs a proof
> that the edit did not change the parse, which is incremental parsing by another name", and then
> dismisses them on the scan arithmetic anyway. The right comparison for those was off by seventy
> fold. See the section below.

This corrects a claim made earlier in this document and in the commit that recorded it: the scans
were called the expensive part of a snapshot on the strength of a gap in an attribution, not a
measurement. The gap was the parse.

### The best remaining lever: five helpers rebuild the document once per edit

Timing every rule **net of any parse it triggered** inverts the earlier reading. The lint divides as
10.3s parsing, 9.5s rule work, 0.2s everything else, and the rule work is not spread evenly:

| rule | net | parse |
|---|---:|---:|
| `unordered-list-style` | 1962ms | 0 |
| `strong-style` | 1507ms | 0 |
| `ordered-list-style` | 984ms | 0 |
| `paragraph-blank-lines` | 897ms | 536 |
| `remove-link-spacing` | 855ms | 0 |
| `empty-line-around-code-fences` | 468ms | 0 |
| the other 37 rules together | 1403ms | |

**Five rules are 6.2s of the 9.5s.** An earlier reading of this called the distribution flat with no
hot rule; that was wrong, and the reason is worth remembering: it timed `Rule.apply`, which contains
the parse, so whichever rule happened to trigger a parse looked expensive and the real outliers were
buried.

They have one cause. Each of those helpers calls
`replaceTextBetweenStartAndEndWithNewValue` **inside a loop over positions**, which rebuilds the
whole document per edit:

| `src/utils/mdast.ts` | helper |
|---|---|
| 523 | `makeEmphasisOrBoldConsistent` |
| 701 | `makeSureThereIsOnlyOneBlankLineBeforeAndAfterParagraphs` |
| 740 | `removeSpacesInLinkText` |
| 992 | `updateOrderedListItemIndicators` |
| 1036 | `updateUnorderedListItemIndicators` |
| 1072 | `updateBlockquotes` |

On a document with more than three thousand list items that is quadratic, and it explains why
`unordered-list-style`, which only swaps a bullet character, costs a tenth of the whole lint.

**Two of the six are done**, `updateUnorderedListItemIndicators` and `removeSpacesInLinkText`, and
they were worth 3.7s: the lint went from 20.1s to 16.4s. The bullets are collected as one character
edits rather than whole item spans, so nested items do not overlap, and the link trims are disjoint
by construction.

**The other four read text an earlier iteration rewrote**, which is what makes them quadratic and
also what stops them being batched:

- `updateOrderedListItemIndicators` rereads nested lists whose indicator widths have changed;
- `makeSureThereIsOnlyOneBlankLineBeforeAndAfterParagraphs` has expanded ranges that share blank line
  separators, and scans backwards through text it has already rewritten;
- `makeEmphasisOrBoldConsistent` and `updateBlockquotes` need an inner node rewritten before the
  outer one containing it, which is Trap #3 and cannot be expressed as a non overlapping edit list.

Since then `makeEmphasisOrBoldConsistent` **was** batched, and it was the largest of them at 1.7s
between the two style rules. It only ever rewrites the two delimiter runs, and nested nodes of one
type have **disjoint delimiters** even though the nodes overlap: `*)*g**` gives nodes `[0,6]` and
`[2,5]` whose delimiters are `[0,1]`,`[5,6]` against `[2,3]`,`[4,5]`. Emitting delimiter edits
sidesteps Trap #3 rather than fighting it. The lint went 16.4s to 14.6s.

**`updateBlockquotes` is genuinely sequential** and should stay that way. For style `space` the
expression `/>([^ ]|$)/g` **consumes** the second `>` while inserting the inter-marker space, so one
pass over `>>ab` can only produce `> >ab`; the inner rewrite has to happen first for the outer one
to see `>> ab` and produce `> > ab`. The `no space` path has the same shape with `/>[ \t]+>/g`. That
dependency is already pinned by a regression test, and the prize is 185ms.

**`updateOrderedListItemIndicators` was reformulated and reverted.** The reformulation is sound:
discard list nodes contained in another so the outermost ranges are disjoint, then emit only the
indicator edit rather than rebuilding the list's text, since an inner rewrite changes only digits
and a delimiter and cannot affect indentation, newlines, ordered versus unordered classification, or
the first indicator at a new level. It passed every gate and was byte identical. It also made **no
measurable difference**: 15.0s and 14.4s against a 14.6s baseline.

So the 984ms this rule costs is not the rebuilding. The likely explanation is that the document's
lists are shallow, so the quadratic rarely bites and the time is the line oriented scan over list
text, but **that is unconfirmed** and worth measuring before anyone tries again. The work is kept at
`.sisyphus/plans/ordered-list-batching-wip.patch`.

`makeSureThereIsOnlyOneBlankLineBeforeAndAfterParagraphs` **is done**, and measuring first is what
justified it. Stubbing out its rebuild call and running everything else gave 1ms against 789ms, so
the rebuilding really was all of it: 5,477 paragraph nodes and 1,179 rebuilds per pass. The ordered
list helper had looked identical and measured the opposite, which is why the measurement is worth
the five minutes.

It now emits one edit per **gap**, keyed on the gap's own bounds so two adjacent paragraphs claiming
the same run of newlines emit it once. 14.6s to 13.6s.

**The gaps are taken by reproducing the old expansion character for character, not by scanning for
newlines**, and that distinction is the whole correctness story. The old code advances one character
past the paragraph before consuming newlines; on a CRLF document that character is the carriage
return. A newline-only scan left it in place and inserted before it, so a document ending `\r`
became one ending `\n\n\r`. **None of the 226 corpus documents use CRLF**, so the differential was
empty while the behaviour had changed. Obsidian runs on Windows, so that was real traffic, not a
corner case. There are eight tests for it now.

That is the eighth time in this work that an empty differential was necessary and not sufficient.
**If a change touches line endings, whitespace runs, or document boundaries, write the test before
trusting the corpus.**

Two cautions from the conversions. The edits must be sorted and non-overlapping before applying, and
that has been the unsafe part of several changes here, so assert it rather than assume it. And
nested nodes of one type report overlapping positions, Trap #3, so `makeEmphasisOrBoldConsistent`
and `updateBlockquotes` deliberately rely on rewriting one position at a time in descending order;
converting those two needs care that the others do not.

### What the parsing actually consists of

The 9.2s is entirely `fromMarkdown`, and `fromMarkdown` divides like this on the 866KB document,
best of three:

| | ms | share |
|---|---:|---:|
| `fromMarkdown`, the whole call | 536 | |
| preprocess, tokenise and postprocess, via micromark's stage exports | 408 | **76%** |
| building the mdast tree from the events | 128 | 24% |

So three quarters of it is micromark tokenising, and a quarter is turning the token stream into
nodes. Note micromark's stage exports (`parse`, `preprocess`, `postprocess`) **are** reachable from
the installed package, so measuring this needs no new dependency.

And every one of the 17 parses is of the **whole document**: the sizes are 840KB, 840, 840, 839 and
so on, 14.3MB of parsing for an 840KB file. Nothing parses a fragment.

That prices the two remaining parser-side ideas:

- **Work from the token stream instead of the tree**, which the AST property inventory says is
  feasible but needs the helper layer rewritten: worth the 24%, about **2.2s**.
- **Fewer parses**, which is the batch structure question: each parse is a full document, so every
  snapshot removed is worth about **550ms**. After the cleanup group, what is left is 1 parse before
  the regular rules, 11 in the batch loop and 3 after. Of the batch loop's boundaries only 3 are
  clashes, and a clash inherently needs a fresh snapshot for the rule that has to be re-run, so
  those are not obviously recoverable. The clearest remaining candidate is that `yaml-title` and
  `yaml-title-alias` each parse the whole document for the same first heading, over two snapshots
  that differ only in frontmatter; the body they care about is identical.

Neither is small, and neither is mechanical.

### Where the 17 parses come from

Attributed by phase, with the batch loop traced rule by rule:

| phase | parses |
|---|---:|
| `runBeforeRegularRules` | 1 |
| `runRulesInBatches` | 11 |
| `runAfterRegularRules` | **5** |
| custom regex | 0 |

The batch loop's 11 boundaries are **3 clashes** (`quote-style`,
`remove-space-before-or-after-characters`, `remove-trailing-punctuation-in-heading`) and the rest
the `runsOnItsOwn` guard, which is the yaml rules and `rulesThatMustSeeEarlierWork`.

`LintContext` is **already lazy** — it parses only when a rule asks for an mdast-derived ignore
type, and caches per text — so all 17 are genuine asks, not waste. Two of them are `yaml-title` and
`yaml-title-alias` parsing the whole 840KB document to find the first heading.

`runAfterRegularRules` did not batch at all: twelve rules applied strictly in sequence, five of them
parsing. Three of those five were the mdast consumers `blockquote-style`, `trailing-spaces` and
`consecutive-blank-lines`, and they parsed **three different snapshots**, so sharing one between
them was worth two parses. Done: they and the frontmatter escape rule between them now go through
the main loop's batching, 17 parses to 15, 13.6s to 12.6s.

The barriers before that group were checked rule by rule and are real:

- the title is taken from the first body heading, so it must see the capitalisation rule's work, and
  the two edits are disjoint so no clash check would catch a stale one;
- emptying the aliases removes the frontmatter and trims the body, which can make an indented first
  line parse as a blockquote, so blockquote style has to lead the group;
- the timestamp rule's `alreadyModified` depends on the identity of everything before it.

One thing that came out of it: the frontmatter-only escape rule and the body-only rules **can still
clash**, because edit ranges are widened to whole lines. The existing retry handles it, and it is
exercised by tests rather than assumed.

**All 17 parses were of distinct texts**, checked by exact comparison rather than by hashing. There
is no redundant parsing left to reclaim; every further saving has to come from making rules share.

### Proving an edit cannot need a reparse

The inversion: rather than deciding whether an edit *requires* a reparse, prove that it *cannot*,
carry the previous tree forward with positions shifted, and reparse only when the proof fails. The
worst case is a failed check costing almost nothing; walking ~28,000 nodes is ~3ms against a 550ms
parse.

Measured by classifying each batch transition's combined edits: no character from
`[\n*_`~\[\]()#>+|!\\<$-]` in the removed or inserted text, and no leading or trailing space or tab
where the edit touches the start or end of a line. **5 of 12 transitions pass**, so the ceiling is
about **5 parses, 2.75s of 12.6s**.

A first, sloppier version of the same test passed only 1 of 12, because it treated `.`, `"`, `'`,
`:`, `=`, `{` and `}` as structural and bailed at any line edge regardless of whitespace. None of
those carry structure inside a line. **The measured size of this prize moved by 5x purely on how
carefully the condition was written**, which is a warning about trusting the first number.

The condition as written is **not yet sound**: an edit that removes all the non-whitespace content
from a line makes it blank, which splits a paragraph, with no structural character involved. Other
candidates needing a verdict are setext underlines, table delimiter rows, link reference and
footnote definitions whose labels are matched elsewhere, lazy continuation inside blockquotes and
list items, opaque content in fenced code, HTML blocks and frontmatter, and tabs versus spaces.

**Closed: 0 of 12 transitions survive a sound condition.** The loose test's 5 accepts all fall, and
the result holds under every looser reading considered, applied together.

The blacklist approach cannot be made sound at all: CommonMark recognition is contextual and
sometimes document-global. A reference definition's label decides whether every matching `[label]`
elsewhere is plain text or a `linkReference`, so an edit to one changes the tree far away with no
structural character anywhere near it. Ordinary letters can complete an HTML terminator such as
`</script>`. There is no finite set of characters that makes an edit safe.

The sound replacement is structural rather than lexical: accept an edit only if it is **strictly
interior to a single source-transparent `text` leaf**, with no ancestor among the link, reference,
code, html, yaml, math or heading types, no line ending touched, and no markdown punctuation in the
non-whitespace run around it. Measured against the same 12 transitions, **condition 1 alone rejects
ten of the eleven non-empty ones**.

**The structural reason it fails is worth keeping, because it is not obvious and it is not fixable
by writing a better condition.** A batch can be carried forward only if *every* edit in it is inert,
and the batches are deliberately large — one has 286 edits across ten rules. Batching and inertness
pull in opposite directions: **every rule added to a batch removes a parse but lowers the chance the
whole batch qualifies to skip one.** Having already batched aggressively, we have made the edit sets
precisely the ones least likely to be provably inert. The two optimisations are not complementary;
they are substitutes, and we already took the one that pays.

Also learned, and relevant to any future attempt: **shifting positions would not have been enough
anyway.** The tree carries `value` strings, heading text, definition identifiers, code language and
meta, and task state, all of which would go stale. And `positionsByType` stores the *same*
`Position` objects as the tree, so shifting both would double-apply the delta.

Reparsing only the affected block and splicing was also rejected, and by the stronger argument:
CommonMark block boundaries propagate through lists, lazy continuation, unclosed fences and html,
and document-global definitions, so choosing the block to reparse is harder than this was.

### Measured for the PR: baseline, small documents, memory

All the performance evidence in this document comes from one 866KB document, which is not
representative. Three things were measured before writing a PR.

**The real baseline is `master`, not this branch's starting point.** `37d1f70` is not on master; this
branch began from already-optimised code on `perf/ignore-types-single-pass`. Measured in throwaway
worktrees, one run each:

| commit | what | parses | parsing | lint |
|---|---|---:|---:|---:|
| `437d49d` | `master` | 119 | 181.7s | **337.6s** |
| `3e9a988` | fork point | 119 | 181.0s | 339.5s |
| `37d1f70` | this branch's baseline | 36 | 50.4s | 105.0s |
| `7c853be` | current | 15 | 9.0s | **13.7s** |

**The honest claim is about 25x against master.** Note `37d1f70` measures 105.0s here against the
87.0s recorded earlier in this document: single runs on a loaded machine vary by around 20%, so
quote the ratio, not the seconds.

**Small documents did not regress; they improved.** Over the 226-document corpus, twice each way:
6,152ms at `37d1f70` against 1,537ms now, **4.00x**, with the median document 8.10ms to 4.44ms and
p90 24.73ms to 8.28ms. Documents under 2KB improved 2.2x as a group.

Two tiny documents appeared slower, with "non-overlapping ranges across runs" — an
`insert-yaml-attributes` example of 19 bytes and an `escape-yaml-special-characters` one of 329
bytes. **That regression does not exist.** Re-measured in isolation at n=1600 per document, the new
code is 2 to 3x *faster* on both, with clearly separated distributions, and it replicates with cold
caches:

| document | old median | new median |
|---|---:|---:|
| 19 B | 2.06ms | **0.96ms** |
| 329 B | 3.93ms | **1.29ms** |

The original claim rested on **two runs per document** at millisecond scale. Two runs is not a
measurement, and "non-overlapping ranges" over n=2 means nothing. This is the third time in this
work that a number was reported before it was trustworthy — the others being the sampled hash that
invented a duplicate parse, and the loose inertness condition that overstated its prize fivefold.
**The habit to keep is that a difference is not real until it survives repetition.**

**The corpus has no 2-10KB documents at all** — 225 of its 226 files are under 2KB and the only
larger one is an excerpt of the big fixture. The size range that most Obsidian notes actually
occupy is therefore still unmeasured.

**Memory: both caches are bounded by entry count, not by bytes, and that is a hazard.** Linting the
866KB fixture once retains 290MB now against 476MB at `37d1f70`, so this work improved it. But the
mdast cache holds `maxSize: 200` entries of `{text, ast, positionsByType}` keyed on whole documents.
At the measured retention for this fixture, **200 large entries would retain about 2.35GB**. The
corpus run filled both caches to their limits at a harmless 10MB, because those documents are tiny.
The limit is pre-existing, from `571d90f`, but this work leans on the cache far more heavily.

### What is left

- `move-math-block-indicators-to-their-own-line` - deferred, see the line reasoning above.
- The `empty-line-around-*` family and their `ensureEmptyLinesAround*` helpers. These depend on the
  source lines either side of the target, so add fixtures where the target sits directly against
  every protected multiline construct.
- `move-footnotes-to-the-bottom`, `re-index-footnotes`. These search the whole document and move
  content, so exclude protected occurrences from discovery but keep the ordering logic global.
- `yaml-title`, and the two footnote rules below, are the cheap ones still outstanding.
- `blockquote-style` and `space-between-chinese-japanese-or-korean-and-english-or-numbers` last:
  they build regexes from `IgnoreTypes.*.placeholder` and read placeholder text out of the
  document, so they must be converted before placeholders can be removed. The CJK rule deliberately
  *keeps* spaces around links, wiki links, inline code and inline math, so it needs "is this CJK
  character next to a protected range of one of these four types", not a plain skip.
- Then `ignoreListOfTypes` and the helpers left without callers can go. Note that
  `updateListItemText` and `updateHeaderText` already have no callers in `src/`, but are still
  imported by `__tests__/trailing-spaces.test.ts`, `__tests__/remove-multiple-spaces.test.ts`,
  `__tests__/remove-space-around-characters.test.ts` and
  `__tests__/remove-space-before-or-after-characters.test.ts`, which run the old implementation
  beside the new one and assert they agree. Those are worth keeping until the migration is over, so
  delete the helpers and those comparisons together rather than deleting the helpers and finding
  the tests will not compile.

## How the list style grouping was decided

Masking replaced an ignored section with a **single line token**, and a token on the line under a
list item is lazy continuation, absorbed into that item. So masking joined lists that the source
keeps apart, and `ordered-list-style` kept counting across an ignored section where the real tree
starts again.

Reproducing that was tried and abandoned. A rule of the form "join two lists when the gap between
them is entirely protected" needs a condition for blank lines before the gap, another for blank
lines between two ignored sections, another for `.` against `)`, and there is no reason to think
that list ends. It amounts to reimplementing list continuation by hand.

What settled it: with the grouping the real tree gives and no merge rule at all, **none of the 226
corpus documents change**. The delimiter case shows why. Masking deviates from a real parse in
exactly one way, its placeholder being absorbed as lazy continuation; every other structural rule
it appeared to get right, it got right because it was parsing too. So natural grouping agrees with
masking everywhere except documents built specifically to separate them.

The difference is therefore accepted and named: **numbering restarts across an ignored section**.
It is pinned in `__tests__/ordered-list-style.test.ts` rather than in the corpus, so that the
corpus keeps its property of being byte identical against the linter as it was before this work.

The general lesson, which applies to the conversions still to come: when the index and masking
disagree, find out whether masking's answer came from parsing or from what the placeholder happened
to look like. Only the second kind is an artifact, and only artifacts are safe to drop.

## What this will not achieve

Milliseconds is not reachable. The linter applies 55 transformations in sequence over the whole
document; even with one parse, each rule still does at least one pass over 866KB. Expect low single
digit seconds as the floor, and around 55-60s from this change alone.

Getting below that means abandoning source-preserving lexical rules for an AST transform and print,
which was considered and rejected: mdast is not a lossless concrete syntax tree, several Obsidian
constructs (wiki links, templater commands, tags, custom ignore sections) are not in the tree at
all, and the rules are formatters that care about exact whitespace and delimiter choice. Printing
would reformat unrelated content and require rewriting nearly every rule.

## Known open issue, unrelated

`dedupe-yaml-array-values` appears three times in the registered rule list and runs three times per
lint. Visible in any per-rule profile. Not investigated.
