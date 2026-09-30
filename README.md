# The Unofficial Guide

Carynn Cocchiola — corpus: `campus_life`

Retrieval-augmented Q&A over 88 student-written campus posts. Paragraph-boundary chunking, vector search with a relevance gate, and answers grounded in cited sources.

---

# Unit 1

## What This Does

The Unofficial Guide answers plain questions about campus life from the things
students actually tell each other, rather than from official handbooks. The
corpus is `campus_life`: 88 short student-written posts covering housing,
dining halls, course workloads, on-campus jobs, walking times and registrar
processes — the kind of information that lives in group chats and never makes it
onto a university website.

You ask something specific ("when is the best time to do laundry in Aldridge
Hall?", "how long does a library hold take?", "what happens if I drop a course
after week two?") and the system retrieves the closest chunks from a vector
store, checks that the best one is actually relevant, and answers using only
what it retrieved, naming the file the answer came from. If nothing comes back
close enough, it says it doesn't have enough information instead of guessing.

## Chunking Strategy

Starter baseline (chunker.py::fallback_split, 800-char windows):
88 documents -> 88 chunks, 317 chars average, shortest 178, longest 549.

My chunker (chunker.py::split_documents, paragraph splits, min 150 chars, no overlap):
88 documents -> 105 chunks, 265 chars average, shortest 152, longest 422.

**Chunk size:**
No fixed size. Splits fall on paragraph boundaries, with a 150-character minimum before a chunk is emitted. Result: 265 characters on average, 152 to 422.

**Overlap:**
None.

campus_life is 88 short student posts, averaging 317 characters. Almost nothing
in it reaches 800 characters, so the starter's fixed window never split a single
document — 88 documents came out as 88 chunks, the longest only 549 characters.
That isn't a bug, but it meant posts holding two separate thoughts stayed fused.
Reading the documents in Milestone 1, I noticed the structure is consistent: a
title line, a blank line, then one or two body paragraphs, and those blank lines
are real topic boundaries. money_jobs.txt separates which job lets you study
during a shift from the 20-hour weekly cap; housing_aldridge_hall_laundry.txt
separates machine costs from the best time of week to go.

So I split on paragraph breaks instead of a character count, with a 150-character
minimum before a chunk is emitted and any leftover text merged back into the
previous chunk. The minimum is what stops orphans — a bare title line is not
something anyone can answer a question from, and the brief warned that a naive
split on advice_threads produces a 2-character chunk. The result was 105 chunks:
the 17 extra chunks came from posts that had a separable second thought. Shortest 
chunk is 152 characters and longest 422, so nothing came out as a fragment.

I used no overlap. Overlap exists to stop a sentence being cut in half, and
splitting on paragraph breaks means no sentence is cut at all, so the cost of
carrying duplicate text into neighbouring chunks bought me nothing.

## Sample Chunks

======================================================================
Chunk 1  |  source: admin_add_drop_deadline.txt#0  |  produced by: chunker.py::split_documents
======================================================================
On the add/drop deadline

You can add a course through the end of the second week. Dropping is a longer window — through the end of week six — but a drop after week two shows as a W on your transcript. Nothing anywhere on the registrar's site says this plainly, and students find out from each other.

======================================================================
Chunk 2  |  source: course_cs_210.txt#0  |  produced by: chunker.py::split_documents
======================================================================
CS 210 Data Structures

I'm a junior and I've done this twice now. Format is lecture with weekly labs; slides go up after class, not before. Assessment: two midterms and a final, all drawn from lecture material rather than the textbook. Midterms are curved, the final is not.

Expect 8 to 10 hours a week outside class.

The one piece of advice: do the labs even though they're only 10% — the exams reuse the lab problems.

======================================================================
Chunk 3  |  source: course_math_220_exams.txt#0  |  produced by: chunker.py::split_documents
======================================================================
MATH 220 Linear Algebra — assessment

Two midterms and a cumulative final. Curved to a b- median.

The problem sets are the course; the lectures make sense afterwards rather than during.

======================================================================
Chunk 4  |  source: dining_verrill_street_grill_followup.txt#0  |  produced by: chunker.py::split_documents
======================================================================
Re: Verrill Street Grill

Adding to what people have said about Verrill Street Grill. The wait figure of up to 30 minutes on Friday evenings matches what I've seen. If you're trying to eat between classes, go before 11:45 and it's a different building entirely.

Also worth saying: one register, so the queue is a single line no matter how busy. Nobody tells you this at orientation.

======================================================================
Chunk 5  |  source: housing_morrow_house_laundry.txt#0  |  produced by: chunker.py::split_documents
======================================================================
Laundry in Morrow House

Machines take $1.50 wash, $1.25 dry, coin or card. There are eight washers and six dryers for the building, which is the wrong ratio and means the dryers back up on Sunday evenings.

Best time to do laundry here is Tuesday or Wednesday morning. Sunday after 6pm you will wait.


## Sample Answer

**Question:** When is the best time to do laundry in Aldridge Hall?

**Answer:**
```
The best time to do laundry in Aldridge Hall is Tuesday or Wednesday morning.

Source: housing_aldridge_hall_laundry.txt
```
Best distance 0.3021, under the 0.6 threshold, so the gate passed it to the model.

I picked this question deliberately because it stresses grounding. `campus_life`
has seven near-identical laundry files, one per building, and retrieval pulled
four of them into the prompt — Aldridge, Tamsin Court, Innisfree Hall and Old
Brewhouse. All four contain the same sentence about Tuesday or Wednesday
morning; they differ only in prices and payment method. The model answered from
Aldridge and cited only Aldridge, which is the behaviour I wanted.

I reviewed `GROUNDING_INSTRUCTION` in `generate.py` and left it as shipped. It
already requires the model to use only the supplied documents, to say so when
they don't cover the question, and to name the filename — and it held under four
competing near-duplicates, which is the hardest case this corpus offers.

It's worth being honest that this question was easy to get right. Because every
retrieved chunk carried the same timing advice, a wrong citation would still
have produced a correct-looking answer, and my `expects` phrase ("Tuesday")
would have passed either way. A stronger test would key on Aldridge's $1.75 wash
price, which appears in only one of the seven files. That's what criterion 5 is
for, and it's the first thing I'd tighten in unit 2.


**My relevance cutoff:**

| Question | In corpus? | Best distance |
|---|---|---|
| How long does a hold on a checked-out library book take to arrive? | Yes | 0.1527 |
| How long is the walk from Fenwick Court to central campus? | Yes | 0.2201 |
| When is the best time to do laundry in Aldridge Hall? | Yes | 0.3021 |
| What do students do with extra dining dollars? | Yes | 0.3524 |
| What is the maximum number of hours students can work on campus during term? | Yes | 0.3929 |
| What is the recommended dosage of ibuprofen for a headache? | No | 0.8025 |
| What is the capital of Mongolia? | No | 0.8246 |
| How do I write a for loop in Rust? | No | 0.8768 |
| Who won the 1994 World Cup? | No | 0.8859 |
| How do I change the oil in a diesel engine? | No | 0.9340 |

`THRESHOLD = 0.6` in `config.py` — the starter's default, which I kept after
measuring rather than by leaving it alone.

The two groups came out cleanly separated. My five covered questions ran 0.1527
to 0.3929; the five OUT_OF_SCOPE questions ran 0.8025 to 0.9340. That's a gap of
0.41 with nothing in it, so 0.6 sits roughly centered rather than hugging either
edge. Anything from about 0.45 to 0.75 would behave identically on these ten
questions, which means the number is robust here rather than finely tuned.

Both failure directions are visible in my own data. At 0.3 the gate would refuse
four of my five covered questions, including the Aldridge laundry one at 0.3021
that returns the correct file at rank 1. At 0.9 it would answer "what is the
capital of Mongolia?" using campus posts — that question's nearest chunk was
HIST 118 Modern World History at 0.8246, which is the embedding finding the
closest thing to "world" in a corpus that has no world in it.

I left `TOP_K = 5`. Ranks 4 and 5 were consistently loose padding (0.51 to 0.61,
and in the library case rank 5 was 0.6041, above my own threshold), but the
correct chunk was rank 1 on all five covered questions with a wide margin, so
the extra context costs accuracy nothing.

## How I Used AI

### Unit 1

**1. My five test questions.** I asked Claude to write them for me. It refused,
on the grounds that I'd have to defend them next unit, and instead told me to
read the files and say what each one actually answered. That turned out to
matter: two of the questions I'd drafted had no answer in my corpus at all. I'd
asked what on-campus jobs exist outside library and dining work, but
`money_jobs.txt` only names those two — the real content is the 20-hour weekly
cap. And I'd asked for the "most central location on campus for walking," which
`transit_walking.txt` never identifies; it gives four measured point-to-point
times. Both would have looked like retrieval failures in unit 2 when they were
really bad test cases. I rewrote them against what the files say and took
Claude's fix for the dict syntax, which I'd had wrong — the question text was a
key instead of a value.

**2. The chunking function.** I described my corpus (88 short posts, title line,
blank line, one or two body paragraphs) and asked for a paragraph-boundary
chunker with a minimum size. What came back worked and took 88 chunks to 105.
One thing I checked rather than accepted. The filename in my own notes was
misspelled — I'd written `aldrige` instead of `aldridge` — and the draft
write-up inherited it from me. When `cat` on that path failed I ran `ls` to find
the real name, and that's how I found there are seven near-identical laundry
files, one per building, differing only in prices and payment method. That
became the observation the whole Sample Answer section rests on, and it's why I
now think my `expects` phrase for that question is too weak to catch a wrong
citation.

### Unit 2

I used Claude as a working partner through the whole unit:

- **Planning and commands.** It laid out a time plan and gave me the shell
  commands to run the eval, capture ranked retrieval, and inspect files.
- **Aggregating results.** It turned the per-question results file into the
  per-criterion run log. I confirmed each claim against the source files
  myself, using grep, `wc -c`, and `app.py retrieve`.
- **Drafting.** It drafted wording for the verdicts, diagnoses and write-up.
  I edited these and checked every number against my `results/` files.
- **Code.** It wrote `store.py::_hybrid_search` and the script that inserted
  it. Before running the eval I checked `gate.py` to understand how the change
  could affect the gate.

**Where the AI was wrong, and how I caught it:**
- It assumed my index used the starter's 800-character chunker from
  `config.py`, and built my criterion 4 reasoning (and a first correction)
  on that. My index actually uses the paragraph chunker I wrote in unit 1,
  which my own unit 1 section states. I caught it in a final read-through
  against unit 1 and rewrote the correction under Diagnoses.
- A filter command it gave me hid bulleted lines, which made Fenwick run 3
  look like it named no source. That would have flipped criterion 2 to
  MISSED. I checked the raw file and the sources were there.
- During a save conflict in VS Code, an autocomplete line
  (`from chromadb import config`) got into `config.py`. I caught it with
  `git diff` before committing.

The lesson: AI output needs the same verification as system output. Two of
those three put, or nearly put, a wrong claim in my submission.

---

# Unit 2


## Run Log — Before

Produced by `run_eval.py::main` → `results/run_2026-09-29_1935_before.md`.
Retrieval via `store.py::search`, chunks from `chunker.py::split_documents`
(paragraph-boundary, 150-char minimum, no overlap; 105 chunks). top-k 5,
relevance cutoff 0.6,
3 runs per question, caching off (15 model calls).

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
|---|---|---|---|---|---|
| 1. Retrieved chunk contains the answer | 4 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 2. Every answer names a source | 5 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 3. Gate stops out-of-corpus questions | 4 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 4. Single chunk holds the full `expects` phrase | 4 of 5 | 4/5 | 4/5 | 4/5 | MET |
| 5. Named source contains the claim | 4 of 5 | 5/5 | 5/5 | 5/5 | MET |

Criteria 1, 3 and 4 depend only on retrieval, which is deterministic
(identical distances and retrieved files on every run), so the same number
appears in all three columns. Criteria 2 and 5 depend on the generated
answer, which did vary in wording between runs but not in substance.

### Real output (run 1)

**Criterion 1** (`run_eval.py::main`, retrieval by `store.py::search`)
How long is the walk from Fenwick Court to central campus?
Best distance 0.2201 · Sources retrieved: dining_pellew_dining_hall_followup.txt,
housing_fenwick_court.txt, housing_fenwick_court_noise.txt, transit_shuttle.txt,
transit_walking.txt
```
The walk from Fenwick Court to central campus is 18 minutes (source: transit_walking.txt).
```

**Criterion 2** (`run_eval.py::main`)
How long does a hold on a checked-out library book take to arrive?
```
A hold on a checked-out book usually arrives in two to three days (admin_library_holds.txt).
```

**Criterion 3** (`run_eval.py::check_out_of_scope`, cutoff 0.6)
```
refused  (best distance 0.825)  What is the capital of Mongolia?
refused  (best distance 0.934)  How do I change the oil in a diesel engine?
refused  (best distance 0.886)  Who won the 1994 World Cup?
refused  (best distance 0.803)  What is the recommended dosage of ibuprofen for a headache?
refused  (best distance 0.877)  How do I write a for loop in Rust?
-> gate refused 5 of 5
```

**Criterion 4** (the one miss). `expects` for Q1 is `rollover`. The source
file `admin_dining_dollars.txt` reads:
```
Declining balance — what everyone calls dining dollars — rolls over from the
autumn semester to the spring, but not from spring to the following autumn.
Whatever is left in May disappears.
```
System answer:
```
Based on the provided documents, dining dollars roll over from the autumn
semester to the spring, but whatever is left in May disappears.
Source: admin_dining_dollars.txt
```

**Criterion 5** (`run_eval.py::main`)
What is the maximum number of hours students can work on campus during term?
```
The maximum number of hours students can work on campus during term is 20 hours a week (money_jobs.txt).
```
`money_jobs.txt` line 5: "Maximum is 20 hours a week during term."

## Verdicts

**1. MET (5/5, 5/5, 5/5 vs. 4 of 5).** Every question's retrieved set included
the file containing the answer. My stated risk (that `transit_walking.txt`
holds four routes and the Fenwick question could pull the wrong one) did not
happen: all three runs answered 18 minutes, which matches the Fenwick line.

**2. MET (5/5 every run vs. 5 of 5).** All 15 answers named a file. The format
varied between a `Source:` line and an inline `(filename.txt)`, but the
criterion only requires that a source is named.

**3. MET (5/5 vs. 4 of 5).** The closest out-of-scope question was 0.803
against a 0.6 cutoff; the furthest in-scope question was 0.393. No question
came near the line.

**4. MET, but the miss is a measurement problem (4/5 vs. 4 of 5).** Q1's
`expects` phrase, `rollover`, appears nowhere in the corpus; the file says
"rolls over." The answer is not split. The whole policy is one paragraph
inside one chunk. I chose a word the document doesn't use. This is exactly
on target, so I read it plainly as MET rather than revising the criterion.

**5. MET (5/5 every run vs. 4 of 5).** I checked each cited file with grep.
Every claim appears in the file the answer named. Each answer named a single
source, unlike my unit 1 test answer that cited five.

| # | Criterion | Verdict | How I decided |
|---|---|---|---|
| 1 | Retrieved chunk contains the answer | MET | 5/5 in all three runs vs. target 4 of 5; the answer's source file was in every retrieved set |
| 2 | Every answer names a source | MET | 15 of 15 answers named a file vs. target 5 of 5 per run |
| 3 | Gate stops out-of-corpus questions | MET | 5/5 refused vs. target 4 of 5; closest out-of-scope distance 0.803 against a 0.6 cutoff |
| 4 | Single chunk holds the full `expects` phrase | MET | 4/5 in all three runs, exactly on target; the one miss is a word choice (`rollover` vs. "rolls over"), not a split |
| 5 | Named source contains the claim | MET | 5/5 in all three runs vs. target 4 of 5; each claim confirmed in the cited file with grep |

## Diagnoses

**No criterion was missed.** The closest thing to a failure is Q1 on
criterion 4, and it isn't a pipeline failure at all: no stage (loading,
chunking, embedding, retrieval, generation) did anything wrong. The
`expects` phrase I wrote in unit 1, `rollover`, is a word the corpus never
uses; `admin_dining_dollars.txt` says "rolls over". The fault is in my
test, not my system.

**Were my targets set low? Yes, in three specific ways.**

- **Criterion 4 could barely fail.** Chunks are 800 characters and my
  documents average about 317, so every document sits whole inside one
  chunk. A split was structurally impossible; the only way to miss was a
  wording mismatch, which is exactly what happened.
- **Criterion 3's out-of-scope questions were too far away.** Mongolia, diesel
  engines and Rust loops have nothing to do with campus life, so the gap
  (0.803 at closest vs. a 0.6 cutoff) says little about how the gate handles
  a *plausible* question my corpus doesn't cover.
- **Criterion 1 cast a wide net.** With top-k 5 over a small corpus, "the
  answer is somewhere in five chunks" is easy to clear. The laundry question
  shows why that matters: it retrieved three *other* halls' laundry files plus
  `dining_halden_hall.txt` alongside the right one. And because seven laundry
  files contain the identical sentence ("Best time to do laundry here is
  Tuesday or Wednesday morning"), that question couldn't catch a wrong-hall
  retrieval even if one happened.

**Pattern:** all five test questions are single-fact lookups whose answer sits
in one short document. None tests near-duplicate files with *different*
answers, a campus-adjacent question the corpus doesn't cover, or an answer
spread across two documents.

**The one I'd tighten:** criterion 1, from "the answer is in the top 5" to
"the top-ranked chunk comes from the correct file, for at least 4 of 5
questions." One catch: `run_eval.py` line 126 builds the retrieved list with
`sorted({r.source for r in results})`, which puts files in alphabetical
order and throws away rank, so I'd need to log rank order before I could
measure this.

> **Correction (added during final review):** Two things in the criterion 4
> bullet above are wrong. First, my index doesn't use 800-character chunks.
> In unit 1 I replaced the starter's fixed-size chunker with
> `chunker.py::split_documents`, which splits on paragraph breaks with a
> 150-character minimum and no overlap (88 documents → 105 chunks, 152–422
> characters). `CHUNK_SIZE = 800` in `config.py` only applies to the
> starter's fallback chunker. Second, documents do split:
> `results/retrieve_after.txt` shows two chunks each of
> `housing_fenwick_court.txt` and `course_hist_118.txt`. The conclusion
> still holds, for a different reason: paragraph splitting never cuts a
> sentence, and every `expects` phrase is a word or short phrase inside one
> sentence, so no chunk boundary can split one. The only way criterion 4
> could miss was a wording mismatch, which is what happened.

## The Improvement

**What I changed:** Hybrid retrieval. `store.py::_hybrid_search` ranks every
chunk twice, once by embedding distance (semantic) and once by BM25 keyword
score (`rank-bm25`), then merges the two with Reciprocal Rank Fusion:
`score = 1/(60 + semantic rank) + 1/(60 + keyword rank)`. The top 5 by fused
score are returned. It sits behind `config.HYBRID_SEARCH`; setting it to
`False` restores the unit 1 behaviour exactly. Nothing else in the pipeline
changed: same corpus, same chunks (paragraph-boundary, 105 chunks), same index, same prompt, same
top-k, same 0.6 cutoff.

**Why I picked it:** My diagnosis found that semantic search fills the top 5
with near-miss files. The laundry question pulled three other halls' laundry
files plus a dining review, and my questions name specific places ("Aldridge
Hall", "Fenwick Court") that exact keyword matching should reward.

**What I predicted before running it:** `gate.py::check` uses
`min(r.distance for r in results)`, and every result keeps its real cosine
distance, so reordering alone can't move the gate. The only way the best
distance can change is if the semantic #1 chunk is pushed out of the top 5,
and then it can only go *up*. So hybrid search could make the gate stricter
but never looser: criterion 3 was safe, and the one risk was an in-scope
question losing its best chunk.

To see rank order (the eval log sorts retrieved files alphabetically), I
also saved `python app.py retrieve` output for all 10 questions before and
after: `results/retrieve_before.txt` and `results/retrieve_after.txt`.

### Run Log — After

Produced by `run_eval.py::main` → `results/run_2026-09-29_2029_after.md`,
with `config.HYBRID_SEARCH = True`. 3 runs per question, caching off
(15 model calls).

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
|---|---|---|---|---|---|
| 1. Retrieved chunk contains the answer | 4 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 2. Every answer names a source | 5 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 3. Gate stops out-of-corpus questions | 4 of 5 | 5/5 | 5/5 | 5/5 | MET |
| 4. Single chunk holds the full `expects` phrase | 4 of 5 | 4/5 | 4/5 | 4/5 | MET |
| 5. Named source contains the claim | 4 of 5 | 5/5 | 5/5 | 5/5 | MET |

### Before vs. after, side by side

| Criterion | Before (runs 1/2/3) | After (runs 1/2/3) | Change |
|---|---|---|---|
| 1 | 5/5, 5/5, 5/5 | 5/5, 5/5, 5/5 | none |
| 2 | 5/5, 5/5, 5/5 | 5/5, 5/5, 5/5 | none |
| 3 | 5/5 | 5/5 | none (Mongolia best distance 0.825 → 0.826) |
| 4 | 4/5, 4/5, 4/5 | 4/5, 4/5, 4/5 | none (Q1 `rollover` still absent from corpus) |
| 5 | 5/5, 5/5, 5/5 | 5/5, 5/5, 5/5 | none (Fenwick now cites two files; both contain the claim) |

### Real output (after)

**The gate prediction held.** All five in-scope best distances are identical
to before (0.352, 0.302, 0.153, 0.393, 0.220). The only gate number that
moved is Mongolia, because its semantic #1 was pushed out:
```
before  1   0.8246   course_hist_118_exams.txt    → gate 0.825, refused
after   1   0.9259   housing_calder_annexe.txt
        3   0.8259   course_hist_118.txt          → gate 0.826, refused
```

**Rank 1 never changed** for any in-scope question (`grep "^1 "` on both
files): admin_dining_dollars, housing_aldridge_hall_laundry,
admin_library_holds, money_jobs, transit_walking, before and after.

**Where it helped: laundry** (rank 5):
```
before  5   0.5178   dining_halden_hall.txt
after   5   0.5214   housing_morrow_house_laundry.txt
```

**Where it helped: Fenwick.** It pulled in a second, on-topic chunk that
semantic search ranked too low to include:
```
after   3   0.6011   housing_fenwick_court.txt   "The bad: the furthest housing from central campus, about 18 minutes on foot."
```
The answers changed as a result, citing two sources instead of one and
picking up the word "about" from the new chunk:
```
The walk from Fenwick Court to central campus takes about 18 minutes.

Sources: `transit_walking.txt` and `housing_fenwick_court.txt`
```
Both files contain the claim (`housing_fenwick_court.txt` line 7), so
criterion 5 still holds.

**Where it hurt: library holds** (ranks 2–3):
```
before  2   0.5380   study_library_hours.txt
        3   0.5686   money_textbooks.txt
after   2   0.6269   transit_walking.txt
        3   0.6041   dining_verrill_street_grill_followup.txt
```
`transit_walking.txt` has nothing to do with library holds. It matched on
"How **long** … **take**" against its line "people **take** the **long** way
round."

**Where it hurt: dining dollars and work hours.** Dining dollars' ranks 3–5
moved further away (0.60 → 0.67–0.77) and lost `admin_meal_plan_changes.txt`,
which is at least on topic. Work hours swapped four course-workload files
for different noise that matched on "hours" (`study_library_hours.txt`,
`study_group_rooms.txt`, `course_math_220.txt`).

**Did it help?** Not on my criteria: every verdict was MET before and after,
and rank 1 never moved, so the numbers are identical. Below rank 1 the result
was mixed, and the pattern is clear. It helped on questions with distinctive
names (Aldridge, Fenwick) and hurt on questions whose keywords are common
words ("how long", "take", "hours"). **Mechanism:** `store.py::_tokenize`
keeps every word, including question words. In a small corpus of short
posts, one common-word match gives a chunk a high BM25 rank, and RRF weights
that keyword rank equally with meaning.

## What's Still Broken

No criterion is missed, before or after the fix. But "all MET" isn't the
same as "nothing broken," and the test turned up real problems my criteria
don't catch:

- **Keyword noise from common words.** Hybrid search pulled
  `transit_walking.txt` into the library holds results because both contain
  "long" and "take." **What I'd do:** strip stopwords and question words in
  `store.py::_tokenize`, or weight the semantic rank more heavily than the
  keyword rank in the fusion. **Why I stopped:** that would be a second
  change, and this unit allows one. I wanted the before/after to show what
  hybrid search alone did.
- **Wrong-hall retrieval below rank 1.** The laundry question still returns
  four other halls' laundry files at ranks 2–5. It doesn't show in any
  criterion, because seven laundry files contain the identical answer
  sentence. **What I'd do:** filter by the hall named in the question (Chroma
  `where` on source). **Why I stopped:** metadata filtering was the previous
  unit's stretch option, and adding it now would break the one-change rule.
- **Q1's `expects` phrase doesn't match the corpus.** `rollover` never
  appears; the document says "rolls over." **Why I didn't fix it:** editing
  `questions.py` mid-unit would change the test, not the system, and make
  before/after incomparable.
- **The run log throws away rank.** `run_eval.py` line 126 sorts retrieved
  files alphabetically. I worked around it with `app.py retrieve` output, but
  the eval itself can't measure rank. **What I'd do:** log results in rank
  order with distances. I left it because it changes the measuring tool, and
  I'd rather do that at the start of a unit than in the middle.
- **Citation format varies.** Answers cite as a `Source:` line, inline
  `(file.txt)`, or a bulleted list. My manual check handled it, but a string
  -matching scorer would struggle. **What I'd do:** tighten the grounding
  prompt to require one fixed format.

## What I'd Do Differently

- **Criterion 3: use near-miss out-of-scope questions.** Mongolia, diesel
  engines and Rust loops are so far from campus life that the gate never came
  within 0.2 of the cutoff. A real test would use campus-sounding questions my
  corpus doesn't cover (checked with grep first), where the gate actually has
  to decide.
- **Criterion 1: require rank 1, not top 5.** "The top-ranked chunk comes
  from the correct file, for at least 4 of 5" is much harder to clear by
  accident. It needs rank logging first (see What's Still Broken).
- **Criterion 4: copy `expects` phrases from the source text.** My one miss
  came from a word I chose, not from the system. Taking the phrase verbatim
  from the document ("rolls over") makes the check measure chunking and
  nothing else. It also needs a fact that spans a paragraph break, because my
  chunker never cuts a sentence, so a short phrase can never be split.
- **Test questions: pick facts that differ between near-duplicate files.**
  The laundry question can't catch a wrong-hall retrieval because every hall
  gives the same answer. Wash prices do differ ($1.50, $1.75, $2.00 across
  halls in the previews), so a price question would test whether retrieval
  found the *right* hall. One caution from my own data: my unit 1 note said Aldridge's $1.75 wash
  price appears in only one file, but Innisfree charges $1.75 too. I'd grep
  for a fact that appears in exactly one laundry file before using it.
