# Landing-Zone Occupancy Model — Design Document

> **Status:** Draft
> **Version:** 0.1
> **Last updated:** 6/29/2026
> **Author:** Jackson Kent
> **Related docs:** SkiSafe Hardware Architecture; (future) Sign-Logic Design; (future) System Interface Document

_One sentence, plain language: what is this document and who is it for? Write it last — you'll know it once the rest exists._

---

## How to use this skeleton

This file is scaffolding, not content. Every blockquote (the `>` blocks) is guidance written **to you, the drafter** — leading questions, advice for the hard parts, and traps to avoid. **Delete each blockquote as you replace it with your own writing.** The italic placeholder lines (`_[...]_`) mark where your prose goes. When all the blockquotes and placeholders are gone, you have a finished draft.

A few principles to keep in mind throughout, since you mentioned wanting to learn the craft:

- **Write down what you don't know, not just what you do.** A design doc is as much a map of your open questions as a record of your decisions. Hidden uncertainty becomes an integration bug later; named uncertainty becomes a task.
- **Separate the *what/why* from the *how*.** Requirements (what the model must do, and why) are stable. Design choices (which YOLO variant, which threshold) are revisable. Keep them in different sections so you can change a "how" without disturbing a "what."
- **Write for a reader who isn't you, six months from now.** If a sentence only makes sense because of something in your head right now, it will fail its one job.
- **Make it skimmable.** Headings, short paragraphs, a schema table where a schema belongs. A doc nobody can navigate doesn't get read.
- **Date it and version it.** A design doc is a living artifact. Stale docs that don't admit they're stale are worse than no doc.

> **A note on the order you write in.** You do not have to draft these sections top to bottom. Many people find Section 2 (Problem Formulation) and Section 7 (Behavior Requirements) are the real spine — drafting those first often makes the rest fall out. Section 1's non-goals are easiest to write *after* you've drafted everything and seen what crept in. Use the structure to organize, not to dictate sequence.

---

## 1. Purpose, Scope & Non-Goals

> **Why this section exists.** It orients a reader who has never seen the project and, just as importantly, fences off everything this doc is *not* about. The non-goals are what keep this document small — which was your whole reason for starting here.
>
> **Ask yourself.**
> - If a teammate read only this section, would they know what the model does and what it deliberately doesn't?
> - What would a reasonable person *wrongly assume* is in scope? Name those things explicitly as non-goals.
> - Which adjacent docs does this one hand off to, and at what boundary?
>
> **How to approach it.** Draft the non-goals list first. It's easier to say what you're excluding than what you're including, and the exclusions bound every section below.
>
> **Avoid.** Vague scope statements ("handle detection well" — what does that commit you to?). Re-describing the whole system. And watch for scope creep: when you catch yourself documenting hardware or sign behavior in a later section, the fix is usually to come back and tighten the non-goals here.

_[In scope: ...]_

_[Out of scope (and which doc owns it): ...]_

---

## 2. Problem Formulation

Our model must be able to take in a single frame from a live video feed, then create a bounding box if:
- There is a skier in frame
- There is a large object in frame (ski pole/ski)

Then it must output any boxes it creates with their corresponding confidence score. A bounding box must consist of enough data to reconstruct it on an unannotated version of the original frame (the default YOLO format has a center x and y with a width and height, but the ultralytics API allows us to access other formats like x,y,w,h)

The system should err on the side of caution, so any box that fully surrounds the person/object will be considered valid, so long as it does not exceed twice their size (an arbitrary limit).

While running, the model will receive a stream of frames from the live video feed and make a prediction on each one. After it assigns all bounding boxes and confidence scores, it will work on the next frame.

It is possible that many people or objects will be in frame, and a large number of bounding boxes may be difficult to check efficiently. To account for this, logic upstream of the model will pre-process frame data so that only relevant areas of the ski resort (_i.e._ jump landing) will be sent to the model for prediction, in which case we would want to check all boxes anyways.

The model has no part in deciding whether or not the zone is occupied. For now, the model's only goal is to provide the coordinates for a bounding box. Logic downstream of the prediction will determine what signal to send to the sign based on the coordinates of any boxes.

Because this logic is still well above the hardware layer, its behavior will live alongside the model documentation. This logic must be able to take in a bounding box from a stream of bounding box data and determine if the landing zone is clear or not.

This will be done by checking for overlap between any bounding box and a designated landing area. We don't have good data on what confidence threshold to use as a cutoff, so this will be determined experimentally in trial periods.

Future functionality may include sending additional context to first responders (ski patrol), which will require more work from the model. At this stage, we intend to swap to a different protocol for detection if we arrive at this point, or implement additional logic with a second model trained for injury detection. In either case, this is a stretch goal that we will not deal with for now.

---

## 3. Data

> **Why this section exists.** An honest accounting of what the model learns from, and where that data misleads. When your evaluation later produces a confusing result, this is the section you'll reread to understand why.
>
> **Ask yourself.**
> - Which train/test split are you using, and *why that one*? What does a random split hide that a disjoint-course or disjoint-date split would expose?
> - What does the dataset *not* contain that your real deployment will encounter? (Empty landings? Non-athlete people? Objects? Crashes?)
> - How far is the training footage from your actual camera — angle, distance, resolution, lighting? Name the domain gap.
> - Are there licensing or usage constraints on the data, and do they reach your intended use?
>
> **How to approach it.** Write the "limitations" subsection as a warning to a future maintainer. Be concrete and a little pessimistic; you are documenting the edges of what you can trust.
>
> **Avoid.** Glossing over the license — check the actual terms and record any constraint as an open risk rather than discovering it downstream. Presenting the dataset as more representative of your deployment than it is. Defaulting to a random split for a system where generalization is a safety property.

_[Dataset and source: ...]_

_[Split and rationale: ...]_

_[Known limitations and domain gap: ...]_

_[Licensing / usage constraints: ...]_

---

## 4. Frame Preprocessing

> **Why this section exists.** It defines the exact path from a raw frame to a model input, so the pipeline is reproducible and — critically — so training and deployment do the *same* thing.
>
> **Ask yourself.**
> - Are train-time and deploy-time preprocessing guaranteed identical? How will you enforce that, not just intend it?
> - Do you process every frame or sample at some rate? What drives that choice?
> - Do you crop to the landing zone before detection, or detect on the full frame and test the zone afterward? (This decision interacts with Section 2 — keep them consistent.)
>
> **How to approach it.** A numbered sequence of transforms with concrete parameters reads far better than a paragraph. Someone should be able to reimplement it from your list alone.
>
> **Avoid.** Leaving parameters as permanent "TBD." Train/deploy skew — the single most common silent killer of model performance in the field. Sliding label logic into this section; preprocessing is about pixels, not meaning.

_[Transform sequence: ...]_

---

## 5. Model & Training Pipeline

> **Why this section exists.** The reproducibility spine. Enough detail that you — or someone else — could rerun training months from now and land in the same place.
>
> **Ask yourself.**
> - Which base model and pretrained weights, and why those?
> - How are the dataset's annotations converted into the format the trainer expects? (Write this step out — it's where silent, hard-to-spot bugs live.)
> - Which augmentations genuinely help in a snow/flat-light domain, and which might *destroy* the signal you depend on?
> - What are your compute constraints (GPU type, session limits, how a large dataset even gets into the environment), and how do they shape your choices?
> - How will you know a run is actually reproducible — seeds, pinned versions, artifact naming?
>
> **How to approach it.** Give the annotation-conversion step its own subsection. It's small but high-risk, and documenting it forces you to actually understand it.
>
> **Avoid.** Under-specifying to the point where you can't reproduce your own best result. Aggressive augmentations (e.g. heavy color shifts) that may not survive contact with an all-white scene. Treating the format conversion as a black box.

_[Base model and weights: ...]_

_[Annotation conversion: ...]_

_[Augmentation, hyperparameters, environment: ...]_

_[Reproducibility: ...]_

---

## 6. Outputs / Prediction Contract

> **Why this section exists.** This is an *interface*, not a description. Another component — the sign logic — will be built against exactly what you specify here. This section is your half of that contract, and precision now prevents integration bugs later.
>
> **Ask yourself.**
> - What are the exact emitted fields, their types, ranges, and units?
> - What's the update rate / cadence of the output?
> - What does the output look like when the model is *uncertain*, or when something has *failed*? (Don't only specify the happy path.)
> - Could someone build the consumer from this section alone, without asking you a single clarifying question?
>
> **How to approach it.** Write it as a schema — a table of fields, or a typed structure — and include one concrete example payload. Examples catch ambiguities that prose hides.
>
> **Avoid.** Prose hand-waving where a schema belongs. Omitting the uncertain/failure outputs. Assuming the consumer "knows what you mean" — the entire point of a contract is that it doesn't have to.

_[Output schema: ...]_

_[Example payload: ...]_

---

## 7. Behavior Requirements

> **Why this section exists.** This is the spec the model is held to, and it's where safety lives. Section 8 (Evaluation) exists to test the claims you make here, so make them testable.
>
> **Ask yourself.**
> - Under exactly what evidence is a "clear" verdict justified? Be precise — this is the decision that puts a person in the air.
> - Your two error types are not equal. What is the real-world cost of a false "clear" versus a false "occupied"? State the asymmetry explicitly; it should drive everything.
> - What does the model do when it is unsure? Is that behavior fail-safe — does uncertainty resolve toward *hold*, never toward *clear*?
> - How long must a condition persist before the state changes? (A single ambiguous frame should not flip a verdict.)
>
> **How to approach it.** Phrase each requirement as a testable assertion — "The system shall not report CLEAR while any detection above confidence _c_ overlaps the zone" — so Section 8 can check each one directly.
>
> **Avoid.** Requirements you have no way to test. Treating false-clear and false-occupied as equally bad — they are not, and a doc that implies they are has buried its most important decision. Any rule, anywhere, where uncertainty could resolve to "clear."

_[Behavior requirements (as testable assertions): ...]_

---

## 8. Evaluation

> **Why this section exists.** To demonstrate — or honestly fail to demonstrate — that the model satisfies Section 7. Design it to *surface* the dangerous failure, not to flatter the model with a comfortable aggregate number.
>
> **Ask yourself.**
> - At your *actual operating threshold* (not averaged over all thresholds), what is the false-clear rate and the recall on the occupied condition?
> - How do you measure performance specifically on the hardest cases — partially occluded or low-visibility frames, your closest proxy for a downed skier? Does the dataset give you a signal you can slice on?
> - Per-frame metrics or per-sequence? Your decision logic is temporal, so what does evaluating it on isolated frames miss?
> - Which split do you report on, and does it actually test generalization to unseen conditions?
>
> **How to approach it.** Tie each metric back to a specific requirement in Section 7 — evaluation should read as "here's how we check requirement R." Lead with the safety-critical metric, not mAP.
>
> **Avoid.** Reporting only aggregate metrics (mAP, AUC) that average away the one failure you can't afford. Evaluating on a random or already-seen split. Ignoring the hard, occluded cases because they hurt the numbers. Per-frame-only evaluation of a temporal decision.

_[Metrics, tied to requirements: ...]_

_[Evaluation protocol (split, slices, sequence-level): ...]_

---

## 9. Assumptions, Open Questions & Failure Modes

> **Why this section exists.** Making your ignorance explicit — and giving it a resolution path — is the mark of a mature design doc, not a weakness in it. This is the section that turns "things I'm worried about" into tracked work.
>
> **Ask yourself.**
> - What must be true for the model to work that you are currently *assuming* without proof? (Fixed camera? A defined zone? Daytime only?)
> - For each open question, what specifically would resolve it, and who or what owns that resolution? (Borrow the hardware doc's habit of tagging each unknown by what it depends on.)
> - What real-world conditions break the model — whiteout, lens flare, snow on the lens, dusk, a crowd in the zone? For each, is handling it in scope or explicitly deferred?
>
> **How to approach it.** A short table works well: each open question, its dependency, its owner. For failure modes, name the condition, the model-level impact, and the in/out-of-scope decision.
>
> **Avoid.** Hiding unknowns to make the doc look more finished — that just relocates the surprise to a worse moment. Listing failure modes without a disposition; even "out of scope for v1" is a real, useful answer, but it has to be written down.

_[Assumptions: ...]_

_[Open questions (tagged by dependency and owner): ...]_

_[Failure modes (condition, impact, in/out of scope): ...]_

---

_End of skeleton. When every blockquote and placeholder is gone, this is your first complete draft — then revisit Section 1's non-goals and the one-line summary up top with fresh eyes._
