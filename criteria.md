# Acceptance criteria — Provenance Guard

Five criteria that say what "working" means for this service, written in unit 7
**before** you built it.

**All five are yours.** None are given.

An acceptance criterion names a number or something plainly observable. *"The
detector is accurate"* is an opinion — and perfect AI detection isn't possible
anyway, so it's an opinion about something nobody can deliver. *"Of 10 pieces I
wrote myself, at most 1 is labelled high-confidence AI"* is a criterion.

Under each, write a sentence or two on **why that target**. *"I allow at most 1
false positive in 10 because on a writing site a wrong accusation costs more
than a missed detection"* is a real answer.

> Missing your own targets next unit costs you nothing. Setting a target so
> easy you can't miss it does.

---

## Pick numbers you can defend

**Cover at least three of these five areas.** They're prompts, not a form.

| Area | A question it could answer |
|---|---|
| False positives | How often may a human's writing get called AI? |
| Score spread | How far apart should clearly-AI and clearly-human text score? |
| Label coverage | Is every one of your three labels actually reachable? |
| The appeal path | Does an appeal always change status and always get recorded? |
| Bad input | What happens when `text` is missing, empty, or enormous? |

Two things worth knowing before you pick numbers:

- **The expensive mistake is one-sided.** Missing some AI text costs the site a
  little. Accusing a real writer costs that person a lot. A criterion that
  treats both errors as equally bad is describing a system nobody would ship.
- **Unit 8 tests across three trials, and the target has to hold across all
  three.** Your first signal runs a model, so identical input can score
  differently twice. A target of "at most 1 in 10" against trials of 2, 1, 2 is
  a **miss**.

---
## 1. Limit false positives for human writing

Of 10 pieces of writing known to be human-written, no more than 1 should be incorrectly labeled as high-confidence AI-generated in each evaluation trial.

**Why this target:** 
I chose a maximum false-positive rate of 10% because falsely accusing a real writer of using AI can damage their credibility. The system should prioritize protecting genuine writers while acknowledging that AI detection is not always accurate.
---

## 2. Maintain a measurable score difference
Across a test set containing 10 clearly human-written and 10 clearly AI-generated pieces, the average AI-likelihood confidence score for AI-generated writing should be at least 0.30 higher than the average score for human-written writing in each evaluation trial.

**Why this target:** I chose a minimum difference of 0.30 because the system should demonstrate a meaningful ability to distinguish between human and AI writing. This target encourages measurable separation without expecting perfect detection.
---

## 3. Ensure all three labels are reachable
All 3 of the 3 required labels—high-confidence AI, high-confidence human, and unsure—must be reachable through valid confidence scores between 0 and 1. Every score within that range must map to exactly one label.

**Why this target:** I chose complete label coverage because the system must communicate uncertainty rather than forcing every submission into either the human or AI category. All three labels should function correctly.
---

## 4. Ensure appeals are processed successfully
100% of valid appeals submitted with an existing content_id and nonempty reasoning must change the submission's status to under_review, create an appeal record in the audit log, and return a confirmation response.

**Why this target:** I chose 100% because the appeal process is essential for protecting writers who may have been incorrectly classified. Unlike AI detection, recording an appeal is a system operation that should work reliably every time. 
---

## 5. Reject invalid or excessively large submissions
100% of submissions containing missing text, empty or whitespace-only text, or text exceeding 10,000 characters must be rejected with an HTTP 400 response. These invalid submissions must not be processed by the detector.

**Why this target:** I chose a 10,000-character limit to allow reasonably lengthy writing submissions while preventing excessively large inputs from consuming unnecessary resources. Rejecting missing or empty text also prevents the system from analyzing invalid submissions.
---

<!-- ─────────────────────────────────────────────────────────────────────────
     UNIT 8 — read this before you change anything above.

     If a criterion turns out to be BROKEN rather than merely unmet, you can
     revise it, and that earns credit. But never delete or edit the original
     line. Add the revision underneath, like this:

         ## 2. Uncertain cases land in the unsure band

         The system handles uncertain cases well.

         **Why this target:** ...

         > **Revised in unit 8:** Text I judged borderline scores between
         > 0.4 and 0.6 in at least 4 of 5 cases.
         >
         > **Why revised:** "well" gave me nothing to check. I couldn't
         > reach a verdict from it at all.

     That's a revision because the criterion couldn't be MEASURED.

     Lowering a target because you missed it is not a revision, and it costs
     you the point:

         ✗ "At most 1 of 10 of my own pieces flagged as AI" → "at most 3
            of 10", because 1 of 10 turned out to be hard.

     A number you missed stays where it is, gets diagnosed, and gets a fix
     attempted. That's where the points are.
     ───────────────────────────────────────────────────────────────────────── -->
