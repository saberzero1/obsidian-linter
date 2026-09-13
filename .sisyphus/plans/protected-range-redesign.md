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

All byte identical over the corpus throughout.

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
