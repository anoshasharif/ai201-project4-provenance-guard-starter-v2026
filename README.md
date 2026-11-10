# Provenance Guard

> ### 👋 Start here
>
> **New to this repo? Read [RUNNING.md](RUNNING.md) first.**
>
> Once `python test.py` passes, in one terminal:
>
> ```bash
> python app.py
> ```
>
> and in another:
>
> ```bash
> curl -X POST http://127.0.0.1:5000/ping \
>   -H "Content-Type: application/json" \
>   -d '{"message": "hello"}'
> ```
>
> That's a web service. Something sent it text and it sent something back.
> Everything this unit is filling in the middle.
>
> **The rest of this file is your submission.** Fill it in as you go.

---

<!-- ─────────────────────────────────────────────────────────────────────────
     Unit 7 asks for the first five sections. Unit 8 adds the five below.

     Everything is pasted as TEXT. A pasted curl response gets full credit;
     a picture of your terminal gets none.
     ───────────────────────────────────────────────────────────────────────── -->

<!-- ═══════════════════════ UNIT 7 — THE BUILD ═══════════════════════ -->

## What This Does

<!-- The path a submission takes, from arriving to the label a reader sees.
     Six or seven steps — this is the trace you wrote in Milestone 1. -->


Provenance Guard is a text analysis application that examines writing and estimates whether it shows patterns associated with AI-generated or human-written text. It uses two signals and provides a reader-friendly classification. The system also records decisions and allows writers to appeal results.

**How the system works:**

1. A writer submits text and a creator ID through the `/submit` endpoint.
2. The GPT-2 model analyzes the text and calculates a model signal based on perplexity.
3. The stylometry component examines sentence length variation, vocabulary diversity, and punctuation density to calculate a style signal.
4. The system combines both signals using a weighted scoring rule in `scoring.py::combine_signals`.
5. The combined score is converted into one of three labels: human, unsure, or AI, with an explanation that a reader can understand.
6. The submission receives a unique content ID, and the decision, scores, and label are recorded in the audit log.
7. If a writer disagrees with the result, they can submit their content ID and reasoning through `/appeal`. The system records the appeal with an `under_review` status while preserving the original decision.

The `/log` endpoint allows the decision and appeal history to be reviewed. The detector's scores are estimates based on writing patterns, not definitive proof of authorship.

---

## Signals and Scoring

<!-- For each signal: what property it measures, why that might differ between
     human and AI writing, and WHAT IT CAN'T SEE.

     The blind spot is not optional. If you can't name it you don't understand
     the signal yet — and unit 8's attack set is built out of blind spots. -->

### Signal one — the model signal

**What it measures:**
The model signal uses GPT-2 to measure how predictable a piece of writing is. It calculates perplexity and converts it into a score between 0 and 1, where higher scores indicate more AI-like writing.

**Why that might differ between human and AI writing:**
AI-generated writing can contain predictable word choices and sentence structures, while human writing may contain more unusual vocabulary and unexpected phrasing.

**What it can't see:**
Predictability does not prove authorship. Human writing that uses simple language may receive a high AI score, while creative or unusual AI-generated writing may receive a low score. This creates a risk of falsely accusing human writers.

### Signal two — the style signal

**What it measures:**
The style signal measures three writing characteristics: sentence length variation, vocabulary diversity, and punctuation density. These measurements are combined into a score between 0 and 1, where higher scores indicate more AI-like writing.

**Why that might differ between human and AI writing:**
AI-generated writing may have more consistent sentence lengths and repetitive writing patterns. Human writing may show greater variation in sentence structure and vocabulary.

**What it can't see:**
The style signal cannot understand the meaning or authorship of text. Writing style varies between people, genres, and languages. A human writer may naturally use consistent sentences, while AI-generated text can be edited to imitate human writing. Vocabulary diversity also depends on text length.

### The combining rule

<!-- Write the rule in words, then give the numbers. What happens when the two
     signals disagree? -->

**The rule:**
I use a weighted average of the model signal and style signal. Since the model signal showed greater separation between my initial human and AI writing samples, I gave it slightly more weight.

When the two signals differ by more than 0.30, I move the combined score halfway toward 0.50 to reflect uncertainty. This helps prevent either signal from making a strong decision when the two disagree substantially.

The final combined score is limited to the range 0 to 1.

**The numbers:**

Model signal weight: 0.60 (60%)

Style signal weight: 0.40 (40%)

Disagreement threshold: greater than 0.30

Disagreement adjustment: (weighted average + 0.50) / 2

Final score range: 0.00–1.00

**Where it lives:** `scoring.py::combine_signals`

<!-- ⚠️ The grader checks your code against that line. If your rule lives
     somewhere else, say where. -->

**A case where my two signals split, and what my rule does with it:**


For my AI-generated conversational writing sample, the model signal was 0.8022, while the style signal was 0.4972. The difference between the two signals was 0.3050, which exceeded my disagreement threshold of 0.30.

Using my 60% model and 40% style weighting, the initial weighted score was 0.6802. Because the signals disagreed substantially, my combining rule moved the score halfway toward 0.50, producing a final combined score of 0.5901.

This placed the sample in the unsure category rather than confidently labeling it AI-generated. The example shows how my rule reduces overconfidence, but it can also make the detector less decisive.


### Milestone 4: Calibration Results

I tested both the GPT-2 model signal and the stylometry signal on four different writing samples to evaluate how well the combined scoring system distinguishes AI-generated and human-written text.

| Sample | Model Score | Style Score | Combined Score |
|---|---|---|---|
| Clearly AI-generated text | 0.6607 | 0.4442 | 0.5741 |
| Human-written poem | 0.1493 | 0.4186 | 0.2570 |
| Formal human-written academic update | 0.4576 | 0.5002 | 0.4746 |
| AI-generated conversational text | 0.8022 | 0.4972 | 0.5901 |

#### Findings

- The human-written poem received a low combined score of 0.2570, correctly falling into the human category.
- The formal academic update received a combined score of 0.4746, placing it in the unsure category despite being human-written.
- The clearly AI-generated sample received a combined score of 0.5741, placing it in the unsure category.
- The conversational AI-generated sample received a combined score of 0.5901. Its model and style signals disagreed by 0.3050, triggering the disagreement adjustment.

#### Limitations and Blind Spots

The two signals do not always agree. The model signal can strongly favor AI while the stylometry signal remains uncertain. The disagreement adjustment reduces overconfident classifications but can also move AI-generated writing into the unsure category.

Formal human writing can also receive an uncertain score because structured, polished language may resemble AI-generated writing.

These scores are heuristic indicators rather than verified probabilities of authorship. More calibration samples and threshold tuning would be needed before using the system for reliable authorship decisions.

#### Audit Logging

The `/submit` endpoint now calculates the model score, style score, and combined score. All three values are recorded in the audit log and returned in the API response.

The `/log` endpoint was tested successfully and confirmed that the new audit entry contains all three scores.


---

## Label Variants

<!-- The exact text of all three labels, as a reader would see them, and the
     score range each covers.

     Write them for someone who has never heard the word "threshold". -->

| Label | Score range | The exact text a reader sees |
|---|---|---|
| high-confidence human | 0.00–0.35 | We think this text was probably written by a person, but we cannot confirm who wrote it. |
| unsure | Above 0.35 and below 0.65 | We cannot reliably determine whether this text was written by a person or generated by AI. This result is not proof of either. |
| high-confidence AI | 0.65–1.00 | This text shows patterns that may be associated with AI-generated writing, but this result does not prove AI was used. |


**Why I worded the "unsure" one this way:**

I chose this wording because an uncertain result should not sound like an accusation. Someone who wrote their own work might feel frustrated or worried if the system suggests they used AI. The label clearly explains that the detector cannot make a reliable determination and that the result is not proof of authorship.



---

## Sample Run

<!-- One submission and one appeal, pasted as text: the request, the response,
     and the log entries. -->


**A submission**

Request (PowerShell):

```powershell
curl.exe -X POST http://127.0.0.1:5000/submit -H "Content-Type: application/json" -d '{\"text\": \"Artificial intelligence has transformed modern society by improving efficiency, automating repetitive tasks, and providing innovative solutions across industries. These advancements continue to shape the future of technology and communication.\", \"creator_id\": \"test-user\"}'
```

Response:

```json
{
  "confidence": 0.5,
  "content_id": "57b3748a-3f50-40d7-a8ec-f439794ec82c",
  "guess": "unsure",
  "label": "Unable to determine authorship.",
  "model_score": 0.6607,
  "style_score": 0.0
}
```


**An appeal**

```bash
$ curl -X POST http://127.0.0.1:5000/appeal \
  -H "Content-Type: application/json" \
  -d '{"content_id": "...", "reasoning": "..."}'
```

```json

```

**What the log shows afterwards**


```powershell
curl.exe http://127.0.0.1:5000/log
```

```json
{
  "count": 1,
  "entries": [
    {
      "combined_score": null,
      "content_id": "57b3748a-3f50-40d7-a8ec-f439794ec82c",
      "creator_id": "test-user",
      "guess": "unsure",
      "label": "Unable to determine authorship.",
      "model_score": 0.6607,
      "status": "decided",
      "style_score": 0.0,
      "timestamp": "2026-11-04T05:13:47+00:00"
    }
  ]
}
```

### Milestone 5: Appeal Test

I tested the `/appeal` endpoint using a real `content_id` from a previous submission that received an AI classification.

**POST /appeal response:**

```json
{
  "content_id": "39a5a5b3-7bb3-4a14-9de2-768aaa9fc0e7",
  "message": "Your appeal has been received and is now under review.",
  "status": "under_review"
}
```

**GET /log verification:**

The audit log retained the original decision and recorded a separate appeal entry:

```json
{
  "content_id": "39a5a5b3-7bb3-4a14-9de2-768aaa9fc0e7",
  "creator_id": "test-user",
  "event": "appeal",
  "reasoning": "I believe this decision is incorrect because the writing style alone does not prove that AI was used. I would like this submission reviewed.",
  "status": "under_review",
  "timestamp": "2026-11-09T21:00:46+00:00"
}
```

### Milestone 5: Label Testing

I tested all three labels through the `/submit` endpoint.

| Label | Combined Score | Test Result |
|---|---|---|
| Human | 0.2570 | Passed using controlled mock scores |
| Unsure | 0.5901 | Passed using actual model and style signals |
| AI | 0.6664 | Passed using actual model and style signals |

All three labels were successfully returned through the API.

The Human classification was verified using mocked model and style scores because the additional real-text samples did not produce a Human-range result. This confirms the API classification logic but does not establish the detector's accuracy on real human writing.

A human-written poem received an AI classification with a combined score of 0.7023. This demonstrates a false-positive risk and highlights the importance of allowing writers to appeal decisions.

The appeal endpoint was also tested successfully. It accepted a valid content ID, recorded the writer's reasoning, and created an `under_review` entry in the audit log.


**Result:** The appeal was successfully accepted and recorded. The original decision remained in the append-only audit log, while the new appeal entry recorded the `under_review` status.

---

## How I Used AI

**Moment 1 — Implementing the Dual-Signal Scoring System**

- **What I asked for:** I asked ChatGPT to help me implement a second writing-analysis signal using sentence length variation, vocabulary diversity, and punctuation density. I also asked how to combine this signal with my existing GPT-2 model signal.
- **What came back:** ChatGPT provided Python code for the stylometry calculations and a weighted scoring approach using 60% model signal and 40% style signal. It also suggested moving the combined score toward 0.50 when the signals disagreed by more than 0.30.
- **What I changed:** I implemented the calculations in `stylometry.py`, configured the weights in `config.py`, and added the combining rule in `scoring.py`. I tested the signals on AI-generated and human-written samples, recorded the results, and documented limitations such as disagreement between the signals.

**Moment 2 — Implementing Labels and the Appeal Endpoint**

- **What I asked for:** I asked ChatGPT to help me create understandable Human, Unsure, and AI labels and implement an appeal endpoint that allows writers to challenge a classification.
- **What came back:** ChatGPT suggested reader-friendly label descriptions and Python code for `score_to_label()` and the `/appeal` endpoint. It also explained how to validate a content ID and record an appeal using the existing audit functions.
- **What I changed:** I updated `scoring.py` with the three labels and connected them to `/submit` in `app.py`. I implemented the appeal endpoint, tested it using a real content ID, and verified that the audit log recorded the appeal with an `under_review` status. I also documented a false-positive risk after a human-written poem received an AI classification.


<!-- ═══════════════════════ UNIT 8 — THE TEST ═══════════════════════ -->

---

## Rate Limiting

**My limits:** ___ per minute, ___ per day

**Why those numbers and not others:**
<!-- Think about two people: a real writer submitting their own work a few
     times an hour, and a script sending a thousand variations to map your
     thresholds. Your numbers have to be liveable for the first and hostile to
     the second. -->

**What a caller counts as:** <!-- per address, or per creator_id? They fail
differently — one script can look like a thousand callers, and one household
can look like one. -->

**The run of status codes when I pushed past it:**

<!-- python run_attacks.py --flood 15 -->

```

```

---

## Attack Run — Before

<!-- Every attack with its outcome and a held-or-broke mark.
     `run_attacks.py --label before` produces the table. -->



**Ten audit log entries from the run**

<!-- Choose entries that show the interesting failures, not the ten easiest. -->

```json

```

**The single worst outcome** — the one I'd least want to explain to a writer
whose work got caught by it:



---

## Run Log and Verdicts

<!-- Your five criteria across three trials. `run_eval.py --label before`. -->

| Criterion | Target | Run 1 | Run 2 | Run 3 | Verdict |
|---|---|---|---|---|---|
| 1.  |  |  |  |  |  |
| 2.  |  |  |  |  |  |
| 3.  |  |  |  |  |  |
| 4.  |  |  |  |  |  |
| 5.  |  |  |  |  |  |

**Real output from one trial**, pasted as text, naming the file and function
that produced it:

```json

```

**Diagnoses**

<!-- For each miss AND each broken attack: the stage and the mechanism.

     Your service has five stages: the route, signal one, signal two, the
     combining rule, and the label mapping.

     Not a diagnosis:  "The evasion attacks got through."
     A diagnosis:      "Four evasion inputs got through. All four had typos
                        added to AI-generated text. My second signal measures
                        word variety, and typos raise word variety — so it read
                        them as more human. My combining rule weights that
                        signal at 0.6, so it dragged the whole score down."

     Look for a pattern. Five failures on one signal is one problem, not five. -->



---

## The Improvement

**What I changed:**

**Which diagnosis pointed at it:**

### Attack Run — After

<!-- Same format. `run_attacks.py --label after` -->



**Did it help, and how do I know:**

<!-- Widening the unsure band usually fixes one problem and creates another.
     Reporting that trade honestly is worth full credit. -->



---

## What's Still Broken

<!-- For each attack still getting through and each criterion still missed:
     what you'd do, and why you stopped. -->



**The trade I'd make if this ran for real:**

<!-- Two sentences. Fewer wrong accusations means more AI text getting through.
     Say which way you'd go and WHO PAYS FOR IT. -->



<!-- ═════════════════════════════════════════════════════════════════════

     SUBMISSION CHECKLIST — unit 7

       [ ] criteria.md has five numbered criteria, each naming a target
       [ ] Each has a reason underneath
       [ ] POST /submit returns content_id, guess, confidence, label,
           model_score and style_score
       [ ] Two signals that measure DIFFERENT things, each with a named blind spot
       [ ] Three label variants, all reachable, written out in full
       [ ] POST /appeal changes status and writes to the log
       [ ] Signals and Scoring names scoring.py::combine_signals
       [ ] Sample Run: a submission AND an appeal, as pasted text
       [ ] At least four new commits
       [ ] Repository URL submitted — WRITE IT DOWN

     SUBMISSION CHECKLIST — unit 8

       [ ] Rate limiting on /submit, with your numbers justified
       [ ] The pasted run of status codes showing 429
       [ ] Audit log with all nine fields, recording rejections too
       [ ] The attack set run IN FULL, every input with an outcome
       [ ] Ten audit log entries pasted as text
       [ ] Five criteria across three trials
       [ ] A diagnosis for every miss and every broken attack — stage AND mechanism
       [ ] One improvement, with Attack Run — After
       [ ] What's Still Broken, including the production trade
       [ ] At least four new commits
       [ ] The SAME repository URL as last unit

     Do not delete and recreate this repository.
     ═════════════════════════════════════════════════════════════════════ -->

---

📖 **How to run this project: [RUNNING.md](RUNNING.md)**
