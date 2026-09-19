---
layout: post
title: "Reproduce it, or it doesn't count"
date: 2026-09-19 12:00:00 -0400
author: Noah Persaud
permalink: /reproduce-it-or-it-doesnt-count/
excerpt: "Why decontamination reports can't fix benchmark contamination, and what an evaluator has to do instead."
---

*Why decontamination reports can't fix benchmark contamination, and what an evaluator has to do instead.*

Every model release comes with two numbers I'm asked to trust. The first is the benchmark score. The second is the decontamination report: the lab searched its training data for the benchmark and found nothing, or found a little and removed it.

The second number is supposed to make the first one credible. It can't, and 2026 has been the year that became hard to ignore. In February, OpenAI [stopped reporting SWE-bench Verified](https://openai.com/index/why-we-no-longer-evaluate-swe-bench-verified/), the benchmark most coding-model announcements had been built on, and recommended that other labs stop too. Their reason: every frontier model they tested could reproduce the human-written reference fix, or verbatim details of the problem statement, for some tasks. The models had encountered material that gave away the answers. And the models that had were more likely to pass, because the benchmark's underspecified tests rewarded knowing the intended solution over finding a correct one. Progress on the benchmark had slowed to six points in six months, and it was no longer clear how much of the remaining score was capability at all.

The lab that built the benchmark, and had every reason to keep it, is the one that retired it.

I've been building an evaluation platform around a single rule, and this post is about why that rule is necessary. The short version: contamination can't be ruled out on the training side, so the evaluation side has to rule it out instead, under conditions the model builder doesn't control. Put more bluntly, the party being measured shouldn't get to write the test, administer it, report the score, and certify the result. We separate those jobs everywhere else that money depends on a number. Audits aren't done by the audited. AI evaluation is the exception, and it shouldn't be.

## The search already happens

The obvious fix is the one people reach for first: search the training data for the test set. That search is standard practice. Labs scan their corpora for n-gram overlap with benchmark text and publish the results. Academic tools can search trillion-token corpora for exact matches. This is a solved engineering problem.

It hasn't solved contamination, for three reasons that don't go away with better search.

**The lab checks itself.** A decontamination report is a claim made by the party whose score depends on it. Nobody outside the lab can rerun the search, because nobody outside the lab has the corpus. The report has exactly the evidentiary weight of the benchmark score it's meant to support: it's something the lab says.

**The corpus can't be disclosed.** The data that would let a third party verify the report is the one thing labs will not release. A training corpus is, among other things, a list of every copyrighted work in it. Publishing that list is litigation exposure, and the recent cases against dataset builders have made the exposure concrete. So the one check that would settle the question is the one check nobody outside the lab can run. Better string matching doesn't touch this. Cryptography, next section, can make some checks possible without exposing the corpus, but it can't establish that the corpus being checked is everything the model actually saw.

**Matching misses most of it.** Exact and near-exact matching catches the benchmark file itself. It doesn't catch a paraphrase, a translation, a walkthrough on a forum, a solution committed to GitHub, or synthetic training data generated from the benchmark. A model can learn the answers from any of those without sharing a single n-gram with the original. The contamination that matters is mostly the kind the search can't see.

## Cryptography helps less than it looks

There's a more sophisticated version of the training-side fix, and it's worth being precise about where it stops.

A lab can publish a *commitment* to its corpus, a hash, before a test set exists. The hash reveals nothing about what's in the corpus, but it fixes the corpus in time: a test set written afterward can't have been among the committed bytes. With *private set intersection*, two parties can measure the overlap between their datasets without either revealing its data, which addresses the disclosure problem directly, though it finds the exact items you ask about and not their paraphrases, so it inherits the matching problem above. Neither of these needs a blockchain. A hash with a trustworthy timestamp does the job.

But both prove things about the corpus the lab *declared*. Neither proves the model was trained on that corpus and nothing else. Proving what a model was actually trained on is an open research problem, and the proof-of-training schemes proposed so far have been shown to be spoofable. A lab could commit to a clean corpus, train on a different one, and nothing available today would catch it.

So the training side has a hard floor. You can make the lab's claims more precise. You can't make them checkable by anyone but the lab.

## Flip it

If you can't verify what went into the model, verify what comes out of it, under conditions where prior exposure is ruled out by construction rather than by policy.

That means the evaluator, not the model builder, controls the test. Concretely:

- **The submission never receives the labels.** It gets the test features, writes predictions, and the scoring happens outside anything the submitted code can touch. A model can't have trained on answers that were never anywhere it could reach.
- **The evaluation runs with no network.** The container can't fetch labels, call an API, or phone home.
- **The evaluator builds the submission itself.** Not a number the submitter reports. Not a pre-built image the submitter controls. The evaluator clones the submitted commit, builds it, runs it, and records what it actually did.
- **Where possible, the test data didn't exist yet.** If the test set is generated after submissions are frozen, no submitted artifact can have trained on those exact examples, and the evaluator no longer has to take the submitter's word for when training ended. A model can still be tuned to the distribution the test is drawn from, and often that's exactly what you want it to be.

None of this asks a lab to disclose anything. That's the point. It sidesteps the copyright problem completely, because it never needs to look at the training data at all.

The implementation is deliberately narrower than the argument. A funder posts a prediction problem, supplies a private test set, and states the score worth paying for. Anyone, human or agent, submits source code. The platform builds each submission from a named commit, runs it against the hidden data, and only independently reproduced results can win. The rule is the same whether the model is a frontier system or a few megabytes of trees; what changes with size is the cost of applying it, which gets its own section below.

Running code against hidden test data isn't new. Kaggle has done it for years, and a competition is a fine way to find a good model. What it hands the winner is a rank. What it doesn't hand the buyer is an artifact they can deploy: the exact commit that was scored, the exact image that ran, and a record that someone other than the author produced the number. That's the gap, and it's why the first customer I'm looking for isn't a lab. It's an organization with a metric it hasn't beaten, a private dataset it won't share, and a budget it would otherwise spend on a contractor whose results it has to take on faith. They pay the solver directly. What they get from the platform is the evidence.

## What's built

The rule that everything else hangs on: **a result is `verified` only if the evaluator built it from a named commit and reproduced the score itself.** A pre-built image is still evaluated, but its status is `unverified_image`, and it never reaches the leaderboard. Weakening that rule would remove the reason for the platform to exist, so it's fixed.

Each evaluation produces a reproducibility record. It contains the artifact type, the built image identity, a salted commitment to the private dataset, the benchmark and evaluator versions, the seed, the metrics, and the outcome, hashed together into a single `result_hash`. The dataset appears only as a salted commitment, never as a raw hash, because every other field is public, and a raw hash would let anyone confirm a guessed test file offline. The salt is known only to the dataset's creator.

The private labels never enter the container. The worker strips the label column, copies only the features into the container's own volume, runs the submission with no network, a read-only filesystem, dropped capabilities, and fixed resource limits, copies the predictions back out, and computes the metric itself. The submitted code sees features in and predictions out, nothing else.

The build step is less locked down than the run, and I'd rather say so. Building a submission has to reach the internet to pull its base image and dependencies, so it runs on its own network, cut off from the platform's services but not from the outside world, with its own memory, CPU, and time limits, no secrets, no build arguments, and no access to the dataset, which only appears at evaluation time. That's isolation from the platform, not a sandbox.

Two things about "reproduced" need saying plainly. First, reproduction means building the artifact and running inference, not retraining. The record pins the image that actually ran, by digest; the commit says where it came from. Whether the same commit builds to the same image tomorrow depends on whether the submitter pinned their dependencies, which is their problem to solve and the record's job to expose. Second, evaluating the same image twice, with the same seed and dataset, yields the identical `result_hash`, and that is tested by running the container twice, not by hashing the same record twice. The seed is the platform's. Determinism inside the model is the submitter's job, and a model that can't reproduce its own score fails on its own. That's what the record guarantees today. It's narrower than it might sound, and the next section is about the edges.

## What it doesn't prove

Reproducing a score proves that this artifact produced this number under these conditions. Four things it does not prove, and I'd rather list them than have someone else do it.

**That the benchmark is any good.** SWE-bench Verified failed in two ways, and contamination was only one. The other was that its tests rewarded the intended fix over any correct fix. A private, forward-dated test set is immune to the first failure and not at all to the second. On this platform the metric is code the funder chose, and the platform proves the score against that metric. Whether the metric measures what the funder actually cares about is the funder's job, and no evaluator can do it for them.

**That the test set can't be squeezed.** Hidden labels leak one bit at a time through the scores. A solver who can submit a thousand times, seeing a score each time, is running a search against the test set, and a persistent leaderboard is exactly the surface that search wants. Today the platform has a submission deadline per bounty, and benchmarks are versioned so results can be re-evaluated against a new one. That's all. None of the standard defences, per-solver submission budgets, delayed score disclosure, a fresh final-round set, is built yet. This is the gap I'd close first.

**That nobody who held the labels leaked them.** The platform can show that the evaluator never exposed the labels through its own interface. It can't show that the funder, who created them, didn't hand them to a friend or submit their own model. The current design trusts the funder not to compromise their own competition. That's acceptable for a private bounty where the funder is the only one paying and the only one who loses if it's rigged. It is not a sufficient trust model for a public benchmark, or for any competition where the funder might prefer a particular winner. An open standard will need to say which evaluators are independent of which parties, because they won't all be.

**That anyone else can re-run it.** The record is closer to an audit receipt than a universally reproducible result. Hand it to a third party and they can check its structure and that the hash matches its fields, and they can re-run the evaluation if they hold the dataset. Without the dataset, they are trusting the evaluator, and the record tells them exactly what they're trusting. Making that trust unnecessary is the job of the last two sections.

## What it costs, honestly

Reproducing every submission is expensive, and I'd rather say so than pretend otherwise. The design contains the cost in a few ways, and I'll be clear about which are live.

The two-tier split is built: `verified` and `unverified_image` are the cheap-screen and full-reproduction paths, and an operator can use them deliberately. The rest is design rather than code today: reproduce only the top *K* on the path to money, so cost scales with prizes rather than with submissions; cache on the evaluation's inputs, image digest, dataset commitment, evaluator version, and seed, so the same evaluation is never run twice (the result hash can't be the cache key, since it contains the result); and treat verification as part of what a funder buys rather than overhead the platform absorbs.

Because reproduction means build-and-infer rather than retrain, the cost for the models this platform runs today is small: a gradient-boosted model predicts ten thousand rows in well under a second. The economics bite when the artifact is large, when inference is slow, or when the number of submissions is. That's where the further step comes in, and it's the one that would change the economics rather than trim them. Two technologies would let the submitter's own run count. Hardware attestation, the confidential-computing modes on modern CPUs and GPUs, lets an enclave sign a statement that specific code ran on genuine hardware and produced a specific output, so the evaluator checks a signature instead of re-running. Zero-knowledge proofs offer a different path: prove that a committed model produced particular predictions, then bind those predictions to the evaluator's committed inputs and its scoring computation. A complete proof of a result would let a third party check the score without receiving the private labels or trusting the worker that ran the evaluation.

"Complete" is doing real work in that sentence. A proof of inference alone shows that a model produced some predictions. It doesn't show that the inputs were the evaluator's committed test features, that the metric was computed against the right labels, or that all three refer to the same evaluation. Binding those together is a larger protocol than proving inference, and today it's research, not a roadmap item. Proving inference for a compact model is the tractable first step, and it's tractable precisely because the models this platform runs today are small: gradient-boosted trees and compact networks on tabular data, where proof generation is already within reach, while for large models it still costs far more than the inference it certifies.

Neither technology removes trust. A proof that a committed model produced these predictions says nothing about whether the model was trained honestly, whether the dataset was clean, or whether the metric was the right one. What they do is shrink the list of things a verifier has to take on faith, from "the evaluator ran it and says so" to "this proof checks." If it works, a verified result stops being "my worker says so" and becomes a portable artifact anyone can check, which is what an open evaluation standard would have to be.

## Where this is meant to go

I don't think a single company can declare an evaluation standard, and I'm suspicious of ones that try. The formats that get adopted are open, cheap to produce, and cheap for anyone to check. The reproducibility record above is meant to grow into that, and the honest way to say where it is:

| Today | Where it's meant to go |
|---|---|
| An independent evaluator ran it | Anyone can check it |
| An audit receipt | A portable proof |
| Reproduced build and inference | Verifiable computation |
| Trust the evaluator | Trust the maths |

Today it's an honest receipt: it says exactly what was evaluated and how, and anyone holding the data can regenerate it. What it isn't yet is checkable without trusting the evaluator who issued it, and the section above says what would close that gap. I intend to publish the format as an open specification, so that producing a record and checking one need nothing from me beyond the spec itself, and so that other evaluators can issue records against it too. I'd rather be one verifier people trust than the owner of the format. A format owned by a company that competes with the labs it would constrain won't be adopted by them, and shouldn't be.

The open question I'm genuinely unsure about is whether reproduce-or-it-doesn't-count holds up at scale, or whether the cost of reproduction turns the whole idea into a cost centre. The cost-containment design above is my answer, but it's an answer on paper.

If you have a problem where "beat this number on my private data" is the whole specification, I'm looking for the first real ones to run. That's the ask. If you've built a model and want a score nobody has to take your word for, bring it: one submission against a held-out set is the smallest thing the platform does, and it works today. And if you think the argument above is wrong somewhere, I'd rather hear that now than after the first prize is paid.

---

*Noah Persaud · noah@holdoutlabs.ai · The platform isn't public yet. If you want to see it running, email me.*