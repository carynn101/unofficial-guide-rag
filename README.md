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

---

# Unit 2

<!-- These sections get ADDED to what's already above. Don't delete or rewrite
     unit 1 — the point is that someone can see what you said before you knew
     how it went. -->

## Run Log — Before

Produced by `run_eval.py::main` → `results/run_2026-09-29_1935_before.md`.
Retrieval via `store.py::search`, chunks from `chunker.py::split_documents`
(fixed-size, 800 chars, 120 overlap). top-k 5, relevance cutoff 0.6,
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

<!-- For each miss: which stage caused it, and how. The stage alone isn't
     enough — you need the mechanism.

     Not a diagnosis: "Question 3 didn't work."
     A diagnosis:     "Question 3 asks about laundry costs. The answer is in
                       one sentence that got split across two chunks, so
                       neither chunk on its own contains it."

     The five stages: loading → chunking → embedding → retrieval → generation.

     Look for a pattern. If three misses all ask about numbers, that's one
     problem, not three.

     Missed nothing? Say so, then say honestly whether your targets were set
     low, and which one you'd tighten and to what.

     Milestone 3. -->

## The Improvement

**What I changed:**

**Why I picked it:**

<!-- Connect it to a specific diagnosis above in one sentence. If you can't,
     you picked a fix because it sounded impressive. -->

### Run Log — After

<!-- Same format, same five criteria, three runs each.
     `python run_eval.py --label after` -->

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
|---|---|---|---|---|---|
| 1. Retrieved chunk contains the answer | 4 of 5 |  |  |  |  |
| 2. Every answer names a source | 5 of 5 |  |  |  |  |
| 3. Gate stops out-of-corpus questions | 4 of 5 |  |  |  |  |
| 4. | | | | | |
| 5. | | | | | |

**Did it help?**

<!-- Say plainly whether it did, and how you know. If it made things worse,
     say that — a change that backfired, honestly reported, earns full credit
     and is more interesting than one that worked. What matters is that you can
     tell.

     Milestone 4. -->

## What's Still Broken

<!-- For each criterion still missed after your fix: what you'd do about it,
     and why you stopped where you did.

     "I ran out of time" is fine if it's true. Pretending nothing is left is
     not.

     Milestone 5. -->

## What I'd Do Differently

<!-- Knowing what you know now — which of your five criteria would you write
     differently, and why?

     Milestone 5. -->
