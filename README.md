# findex

A search engine, built lab by lab, as part of the *Python — Build a
Search Engine* course. This is **Lab 1**: a corpus loader and
tokenizer that stream text instead of loading it all into memory.

## What's here

```
src/findex/
  corpus.py     iter_documents(root) -> Iterator[Document]   (generator)
  tokenize.py   tokenize(text) -> Iterator[str]               (generator)
  stats.py      chains the two into a one-pass stats pipeline,
                python -m findex.stats data/
tests/
  test_tokenize.py   10 assert-style checks
```

## The corpus

This repo currently ships with a small **demo corpus** (in
`data/`, not committed — see below) made of:
- the first 10 chapters of *Alice's Adventures in Wonderland*
  (Lewis Carroll, public domain, via Project Gutenberg), split one
  chapter per file, and
- three short personal notes in Ukrainian, to prove the tokenizer
  handles Cyrillic and apostrophized words (`п'ять`, `м'ята`)
  correctly.

**This is a placeholder.** Before submitting, replace `data/` with
your own real corpus — your notes, a Project Gutenberg collection,
a Wikipedia dump slice, arXiv abstracts, whatever you'll enjoy
searching — growing toward ~1,000 documents / ~50 MB by the end of
the course. `data/` is in `.gitignore`, so it never gets committed;
the README should say where the corpus comes from and how to get it.

**Where to get one:** the lab handout lists Project Gutenberg,
the Simple English Wikipedia dump, arXiv abstracts, 20 Newsgroups,
or your own notes. Pick whichever you'll enjoy searching.

## Running it

```bash
uv sync --extra dev            # or: pip install -e ".[dev]"
python -m findex.stats data/                # full corpus, streaming
python -m findex.stats data/ --limit 5      # just the first 5 docs
python tests/test_tokenize.py               # run the tokenizer checks
```

## Tokenizer policy

- **NFC-normalize** first (`unicodedata.normalize("NFC", text)`), so
  `café` written as one code point and `café` written as `e` + a
  combining accent become the *same* token.
- **`casefold()`, not `lower()`** — the aggressive, locale-independent
  form (`"Straße".casefold() == "strasse"`).
- Tokens matched with **`re.finditer`**, not `re.findall` — matches
  are produced lazily, one at a time.
- **Apostrophes** (`'` and the Unicode `’`) are kept *inside* a word
  (`don't`, `п'ять`) but stripped at the edges (`'hello'` → `hello`).
- **Hyphens** are kept inside a word (`well-known`) but not as
  leading/trailing punctuation.
- **Digits** are matched by `\w` like any other word character, so
  numbers become their own tokens (`"2024"` → one token); no further
  special-casing in this lab.
- `\w` is Unicode-aware in Python 3 by default, so Cyrillic and
  accented Latin tokenize correctly with no extra flags.

## M4 — eager vs. lazy measurement

`stats.py --eager` runs a deliberately greedy version: it reads
every document into a list, tokenizes every document into a list of
token-lists, flattens that into one giant list, and *only then*
builds the `Counter`. `stats.py` (default, no flag) runs the
streaming pipeline: `Counter(tok for doc in docs for tok in
tokenize(doc.text))`.

Both were run on the same corpus slice — the demo corpus above,
duplicated to 400 documents (~340 KB of Alice-in-Wonderland text) so
the difference is visible without exhausting a laptop:

| Version           | Documents | Peak memory | Elapsed |
| ------------------ | --------- | ----------- | ------- |
| eager (lists)      | 400       | 4426.7 KiB  | 0.1698 s |
| lazy (generators)   | 400       | 344.1 KiB   | 0.1624 s |

The eager version's peak memory is about **13x** the lazy version's,
for identical output (same token count, same vocabulary, same
counts).

**Why:** the eager path builds and holds four separate full copies
of the corpus's *tokens* in memory at once — the list of `Document`
objects (each holding a whole file's text), the list of per-document
token lists, the flattened list of every single token, and finally
the `Counter` built from that flattened list. Each of those is
proportional to total corpus size. The lazy path never has more than
one `Document`'s text and a handful of in-flight tokens alive at a
time — the *only* thing that legitimately grows with the corpus is
the `Counter` itself, because it holds the answer. That's also why
the lazy version's memory isn't literally zero: the vocabulary
(`Counter` keys) and their counts still have to live somewhere, and
that's the correct, necessary memory to spend.

On the real, full-size corpus (once `data/` is your own ~1,000
documents / ~50 MB), re-run both commands and replace the numbers
above with your own — the ratio should hold or grow, since the eager
version's overhead scales with corpus size while the lazy version's
extra memory stays roughly flat.

## Definition of done (checklist)

- [x] `iter_documents` and `tokenize` are generators
      (`next()` on `iter_documents` returns instantly).
- [x] `python -m findex.stats data/` runs in one pass, streaming.
- [x] eager-vs-lazy table above, with real numbers.
- [x] tokenizer policy documented and tested (10 checks, incl.
      Cyrillic and a combining-mark case).
- [ ] `data/` replaced with your real corpus before submitting.
- [ ] tag `lab-01` pushed (then `lab-02` after Lab 2).

---

# Lab 2 — The inverted index

```
python -m findex.index data/ --out index.bin                 # build + save (pickle)
python -m findex.index data/ --out index.fbin --format binary  # build + save (own array format)
python -m findex.index data/ --out index.bin --positions      # also store token positions
python -m findex.search index.bin "alice rabbit"              # AND
python -m findex.search index.bin "queen OR duchess" --engine set
python -m findex.search index.bin "alice NOT rabbit"
python -m findex.bench index.fbin                             # merge vs set
python -m findex.memstudy data/                               # the M4 table
python tests/test_search.py
```

Files: `model.py` (`Posting`, `DocMeta` = `@dataclass(frozen=True, slots=True)`, `Index`),
`index.py` (one-pass build with `defaultdict`), `search.py` (two-pointer merge + set engine),
`store.py` (pickle + own binary format), `bench.py`, `memstudy.py`.

Query language: left to right, `a b` = AND, `OR`, `NOT` (= AND NOT). Operators are UPPERCASE.

## Measurements

> The numbers below were measured on a **synthetic Zipf-distributed corpus**
> (2000 documents x 300 tokens, 20 000-word vocabulary: 402 518 postings,
> 21 726 terms), because the demo corpus is tiny. **Re-run the commands on your
> own corpus and replace the numbers** before submitting.

### Postings storage (M4)

| Postings representation         | Peak memory (build) | Pickle size | Load time |
| ------------------------------- | ------------------- | ----------- | --------- |
| `list[Posting]`, plain dataclass | 43.7 MB            | 9.18 MB     | 0.332 s   |
| `list[Posting]`, `slots=True`    | 30.8 MB            | 6.76 MB     | 1.255 s   |
| `array('I')` pairs (no objects)  | 11.7 MB            | 4.25 MB     | 0.027 s   |

Where the bytes went (about 400 k postings): a plain dataclass instance carries an
object header plus a per-instance `__dict__` machinery (~100 B per posting in total
here). `slots=True` removes the `__dict__`, so each posting is a fixed 56-byte object
(~74 B per posting including the list pointer) — about 30 % less. `array('I')` stores
each doc_id and tf as a raw 4-byte integer, with no object per posting: 8 B of payload
per posting (the rest of its ~26 B is per-term overhead: 21 k small arrays + tuples +
dict entries). That is ~3.7x less than the plain dataclass.

The slotted row loads *slower* (1.26 s vs 0.33 s): unpickling a frozen+slots dataclass
rebuilds each object through `object.__setattr__`, which is slow. So `slots` wins on
memory, not on pickle-load time. The array format wins on both.

### Positions flag (cost)

| Storage        | Peak memory, no positions | with `--positions` | Pickle size (no / with) |
| -------------- | ------------------------- | ------------------ | ----------------------- |
| plain dataclass | 43.7 MB                  | 67.2 MB (+54 %)    | 9.18 / 10.90 MB         |
| slots           | 30.8 MB                  | 54.3 MB (+76 %)    | 6.76 / 8.48 MB          |

(The array format does not store positions.) On real text with repeated words the
positions cost is larger; here most terms occur once per document.

### Save / load, two formats

| Format                       | File size | Save time | Load time |
| ---------------------------- | --------- | --------- | --------- |
| pickle (slots postings)      | 6.76 MB   | 1.434 s   | 1.126 s   |
| own binary (JSON header + `array('I')`) | 3.83 MB | 0.036 s | 0.036 s |

### Boolean AND: merge vs set

Two most common terms: `doc` (df 2000), `w0` (df 2000). Two rarest: `w9954`, `w9960` (df 1).

| Case              | merge    | set      |
| ----------------- | -------- | -------- |
| common AND common | 198.1 µs | 99.2 µs  |
| rare AND rare     | 0.1 µs   | 0.3 µs   |
| common AND rare   | 115.3 µs | 22.3 µs  |

Result: `set` wins on long lists because `set(a) & set(b)` runs in C, while our merge
is a Python-level `while` loop (both are O(n+m)). `merge` wins only when both lists are
tiny (no set construction overhead). The merge's real value is not raw speed in CPython:
it streams, needs no extra memory, works on sorted/compressed postings, and is the base
for phrase queries in Lab 3.

### Security note

`pickle.load` can execute arbitrary code from the file, so never load a pickle you did
not create yourself. The binary format only parses JSON and copies bytes, so it is safe.

---

# Lab 3 — Ranking and the object model

```
python -m findex.index data/ --out index.bin --positions     # positions are needed for "phrases"
python -m findex.ranking index.bin 'queen OR duchess' -k 5    # ranked, with snippets
python -m findex.ranking index.bin '"cheshire cat"' -v --repeat 2   # @timed + cache log
python -m findex.ranking index.bin 'alice AND (queen OR duchess) NOT rabbit' --scorer tfidf
python -m findex.sanity index.bin                            # 3 ranking sanity checks
python -m findex.evaluate index.bin queries.json             # precision@5, TF-IDF vs BM25
python tests/test_lab3.py
```

New files: `query.py` (recursive-descent parser + `Term/Phrase/And/Or/Not` nodes with `&`, `|`, `~`),
`scoring.py` (`Scorer` protocol, `TfIdf`, `BM25(k1=1.5, b=0.75)`), `ranking.py` (`Searcher`,
`SearchResult(order=True)`, top-k with `heapq.nlargest`), `snippets.py`, `utils.py` (`@timed`),
`sanity.py`, `evaluate.py`, `queries.json`. `Index` (in `model.py`) is now a
`collections.abc.Mapping`: `len(index)`, `"term" in index`, `index["term"]`, `for t in index`,
`repr(index)`, `num_docs`, `avg_doc_length` (`cached_property`), `df`; `store.open_index(path)`
is a `@contextmanager` that calls `index.close()` in `finally`.

Query language: `a b` (AND), `AND`, `OR`, `NOT`, parentheses, `"exact phrase"`. Operators are UPPERCASE.
AND binds tighter than OR: `a OR b c` is `Or(a, And(b, c))`.

## Three ranking sanity checks (`python -m findex.sanity`)

Measured on the demo corpus (13 documents, avg length 130.3 tokens), BM25 with k1=1.5, b=0.75:

1. **Rare beats common** (same document, tf=1): `__next__` (df=1) scores **2.429**, `to` (df=10) scores **0.313**.
2. **Saturation** (median-length document, rare term): tf=1 → 2.429, tf=2 → 3.385, tf=10 → 4.942,
   tf=20 → 5.243, tf=21 → 5.259. The 2nd occurrence adds **0.956**, the 21st only **0.0153**.
3. **Length normalisation** (one match): the shortest document (41 tokens) scores **3.230**,
   the longest (559 tokens) only **0.900**.

## Logs: `@timed` and the query cache (`-v --repeat 2`)

```
[findex.timing] load took 2.60 ms
[findex.timing] Searcher.search took 0.17 ms
[findex.search] cache hit for '"cheshire cat"'
[findex.timing] Searcher.search took 0.15 ms
```

`lru_cache` wraps the "query -> matched doc_ids" step (one cache per `Searcher`, because an `Index` is
not hashable and `lru_cache` needs hashable arguments).

## precision@5, TF-IDF vs BM25 (`python -m findex.evaluate`)

10 hand-labelled queries on the demo corpus (labels in `queries.json`; multi-word queries are run as OR).
P@5 = (relevant documents in the top 5) / 5, so a query with only 1 relevant document can score at most 0.20.

| Query | relevant | P@5 TF-IDF | P@5 BM25 |
| ----- | -------- | ---------- | -------- |
| grow shrink size | 4 | 0.60 | 0.60 |
| tea hatter riddle time | 1 | 0.20 | 0.20 |
| croquet flamingo hedgehog | 1 | 0.20 | 0.20 |
| mock turtle school lessons | 2 | 0.40 | 0.40 |
| caterpillar mushroom | 2 | 0.40 | 0.40 |
| duchess baby pig pepper | 1 | 0.20 | 0.20 |
| pool tears mouse | 2 | 0.40 | 0.40 |
| queen head executed | 2 | 0.40 | 0.40 |
| rabbit gloves fan | 2 | 0.40 | 0.40 |
| генератор yield | 2 | 0.40 | 0.40 |
| **mean** | | **0.36** | **0.36** |

Both scorers retrieve every relevant document here, so the table is a tie. That is expected on a
13-document corpus of short chapters; the difference between the two scorers appears on larger corpora with
documents of very different lengths. **Replace `queries.json` and this table with your own corpus.**
