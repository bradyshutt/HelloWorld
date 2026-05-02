# Narrative AI Prompting

## Summary

Retelling a user's mundane day as an epic saga, noir, or sportscaster bit is a *style transfer* problem layered on top of a *faithfulness* constraint: the LLM must amplify voice and structure while preserving who, what, when, and where. The 2025 literature converges on a small set of reliable levers — explicit persona/style sheets, 1-3 high-quality few-shot exemplars, structural separation of "facts" from "telling," and post-hoc factuality checks (FactScore / QAG-style atomic-fact verification). For a voice journal, the safest pattern is a two-stage pipeline: (1) extract a structured fact ledger from the transcript, (2) rewrite in the chosen style using that ledger as a hard constraint, then verify the rewrite's atomic claims against the ledger before playback.

## Style Transfer Techniques

- **Persona / role prompts.** Assign a narrator identity ("You are a gravelly 1940s noir detective narrating the user's day in first person"). Persona prompts steer lexical choice, sentence rhythm, and figurative density without needing fine-tuning, but they leak persona-specific *facts* unless constrained — see the persona-faithfulness work in the role-play survey ([Neph0s/awesome-llm-role-playing-with-persona](https://github.com/Neph0s/awesome-llm-role-playing-with-persona)).
- **Style sheets / rubrics.** Spell out diction, sentence length, POV, tense, taboo words, and 3-5 signature devices (e.g., "use weather as mood; address the reader once per paragraph"). Anders Ohrn's writeup on stylish LLM writing recommends extracting an explicit style rubric and reusing it as context ([Towards AI: How To Make LLMs Write Stylishly](https://pub.towardsai.net/how-to-make-llms-write-stylishly-6691be12b970)).
- **Few-shot exemplars.** The ACL "Recipe for Arbitrary Text Style Transfer" shows that 2-3 paired (neutral → styled) examples beat long prose instructions for arbitrary styles ([ACL 2022 Recipe](https://aclanthology.org/2022.acl-short.94.pdf)). Amazon Science's "Conversation Style Transfer using Few-Shot Learning" extends this to dialogue and recommends a *neutralize-then-restyle* two-step that prevents the model from copying example content ([Amazon Science PDF](https://assets.amazon.science/2e/13/09db2e194e01ac743a2767b5c703/conversation-style-transfer-using-few-shot-learning.pdf)).
- **Dual-layer (sentence + paragraph) structure mapping.** For long retellings, extract structure (acts, beats) separately from sentence-level style ([arXiv 2505.07888](https://arxiv.org/html/2505.07888v1)).
- **Imitating a specific person's voice.** Prompt-only imitation works when you supply a small "voice fingerprint" of catchphrases and syntactic tics ([arXiv 2410.03848](https://arxiv.org/html/2410.03848v1)).

## Faithfulness vs Creativity

The 2025 hallucination literature distinguishes *factuality* errors (wrong about the world) from *faithfulness* errors (wrong about the source) — for a journal, faithfulness is the priority ([Frontiers in AI 2025 survey](https://www.frontiersin.org/journals/artificial-intelligence/articles/10.3389/frai.2025.1622292/full); [Lakera 2026 guide](https://www.lakera.ai/blog/guide-to-hallucinations-in-large-language-models)). Effective mitigations that transfer to creative rewrites:

- **Fact-ledger grounding.** Pre-extract a JSON list of events, people, places, times from the transcript and pass it as a "non-negotiable" block. The model may embellish *manner* but not *matter*.
- **Decoupled creativity budget.** Tell the model exactly which dimensions are creative (metaphor, pacing, vocabulary) and which are locked (names, numbers, sequence, outcomes). Lakera and Master of Code report 60-80% error reductions when prompts explicitly partition mutable vs. immutable content ([Master of Code](https://masterofcode.com/blog/hallucinations-in-llms-what-you-need-to-know-before-integration)).
- **Self-check / multi-candidate selection.** The ACL Findings 2025 result shows generating N candidates and picking the most faithful via a lightweight scorer beats single-shot generation without retraining.
- **Atomic-fact verification.** FactScore and QAG decompose the rewrite into yes/no questions answered against the ledger — a natural fit for offline post-processing in a journal app ([Confident AI evaluation guide](https://www.confident-ai.com/blog/llm-evaluation-metrics-everything-you-need-for-llm-evaluation); [Turing factuality guide](https://www.turing.com/resources/llm-factuality-guide)).
- **Tolerance tuning.** A creative tool can accept some imaginative deviation that a financial assistant cannot — set a per-style threshold (e.g., fairy-tale allows added sensory detail; courtroom-drama does not).

## Sample Personas / Styles

- **Epic saga / Tolkienesque bard** — third-person omniscient, archaic diction, name-as-epithet ("Maya, Slayer of the 9 a.m. Standup"), invocation opening.
- **Noir detective** — first-person past, terse sentences, weather + cynicism, internal monologue, one extended metaphor per scene.
- **Sportscaster play-by-play** — present tense, second-person color commentator interjections, stat callouts ("that's her third coffee of the quarter, folks").
- **Fairy tale** — "Once upon a time," rule of three, moral coda, animal helpers standing in for coworkers.
- **Nature documentary (Attenborough)** — hushed observational present tense, biological framing of human behavior.
- **Ship's-log / captain's log** — stardate-style timestamping, clipped declaratives, faithful to chronological order — a good *low-creativity* default.

## Recommendations

1. **Two-stage pipeline.** Stage 1 extracts a structured ledger (events, entities, times, emotions). Stage 2 rewrites from the ledger in the chosen style. This is the single highest-leverage choice for faithfulness.
2. **Style packs as reusable assets.** Each style = {persona line, 5-bullet style rubric, 2 few-shot pairs, locked-fields list, creativity budget}. Store as data, not code, so users can author their own.
3. **One-shot beats zero-shot, three-shot rarely beats two.** Curate 2 short paired examples per style; longer prose instructions degrade adherence.
4. **Verify before TTS.** Run a QAG-style atomic-fact check of the rewrite against the ledger; if any locked field fails, regenerate with a stricter prompt rather than ship a fabrication.
5. **Expose a "creativity dial."** Map it to temperature *and* to how many style devices the rubric permits, so users trade fidelity for fun explicitly.
6. **Log user edits as preference signal.** Edits where users restore a fact are gold-standard faithfulness data; edits where they soften flourish are style-preference data — keep them separate.

Sources:
- [A Recipe For Arbitrary Text Style Transfer with Large Language Models (ACL 2022)](https://aclanthology.org/2022.acl-short.94.pdf)
- [Conversation Style Transfer using Few-Shot Learning (Amazon Science)](https://assets.amazon.science/2e/13/09db2e194e01ac743a2767b5c703/conversation-style-transfer-using-few-shot-learning.pdf)
- [Long Text Style Transfer via Dual-Layered Structure Extraction (arXiv 2505.07888)](https://arxiv.org/html/2505.07888v1)
- [Imitating a Real Person's Language Style with Prompts (arXiv 2410.03848)](https://arxiv.org/html/2410.03848v1)
- [Survey on LLMs for Story Generation (EMNLP 2025 Findings)](https://aclanthology.org/2025.findings-emnlp.750.pdf)
- [Awesome LLM Role-Playing with Persona](https://github.com/Neph0s/awesome-llm-role-playing-with-persona)
- [Hallucinations in LLMs: Frontiers in AI 2025 survey](https://www.frontiersin.org/journals/artificial-intelligence/articles/10.3389/frai.2025.1622292/full)
- [Lakera guide to LLM hallucinations (2026)](https://www.lakera.ai/blog/guide-to-hallucinations-in-large-language-models)
- [Master of Code: reducing hallucinations 60-80%](https://masterofcode.com/blog/hallucinations-in-llms-what-you-need-to-know-before-integration)
- [Confident AI: LLM Evaluation Metrics Guide](https://www.confident-ai.com/blog/llm-evaluation-metrics-everything-you-need-for-llm-evaluation)
- [Turing: Factuality in LLMs](https://www.turing.com/resources/llm-factuality-guide)
- [Towards AI: How To Make LLMs Write Stylishly](https://pub.towardsai.net/how-to-make-llms-write-stylishly-6691be12b970)
