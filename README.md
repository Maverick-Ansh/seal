# seal

Rebuilding **SEAL: Self-Adapting Language Models** from first principles on a free Colab T4.

Paper: Zweiger, Pari, Guo, Akyürek, Kim, Agrawal. *Self-Adapting Language Models.* [arXiv:2506.10943](https://arxiv.org/abs/2506.10943), NeurIPS 2025, MIT.

This is an experiment, not a library. Everything lives in one annotated Colab notebook that is meant to be read top to bottom.

## The idea in one paragraph

A language model is frozen after pretraining. It cannot study a new document the way a person can. SEAL asks what happens if the model writes its own study notes about a new passage, finetunes itself on those notes, and then gets graded on questions about the passage with the passage taken away. The notes are called a **self-edit**. If the updated model answers better, that self-edit was good, and a reinforcement learning loop reinforces the act of writing that kind of note.

Two nested loops:

```
OUTER LOOP (reinforcement learning, slow)
  teaches the model HOW TO WRITE a good self-edit
  |
  +-- INNER LOOP (supervised finetuning, fast)
        applies one self-edit, producing a temporarily updated model
        that model is graded, and the grade is the reward
```

The reward is not a human preference and it is not a regex on an answer. The reward is the downstream accuracy of a whole finetuned model. That is what makes the method expensive and interesting.

## Everything is written out, nothing is imported

- LoRA is hand written, about thirty lines, with five unit tests that assert the paper's rules rather than tensor shapes
- The inner loop is hand written, so attach, train, measure and discard is explicit
- The graders are hand written, including the official SQuAD normalisation
- No `peft`, no `trl`

## How it was resized

|  | Paper | Here |
|---|---|---|
| GPU | 2x H100 or H200 | 1x Tesla T4, free tier |
| Base model | Qwen2.5-7B | Qwen2.5-0.5B-Instruct |
| One RL round | about 6 hours | target about 20 minutes |
| Grader | GPT-4.1 through the OpenAI API | deterministic string matching, built and validated in the notebook |

Shrinking the base model is defensible because the paper holds it fixed across every row of its own Table 2, and because Appendix B.7 already runs this experiment at 3B and 7B and reports that the effect shrinks with model size. Running at 0.5B extends that table one step further down. The thing under test, which is the self-edit policy and the RL loop, is not resized at all.

## What has been found so far

**The evaluation broke twice before the method was ever run.** Both times the bug would have produced a confident wrong conclusion.

1. **Verbosity was being scored as failure.** The first measurement said the model scored 11.6 percent with the passage sitting in front of it. That looks like a model too small for the task. It was actually an Instruct model answering in full sentences while being graded by a metric that demands a bare noun phrase. The paper used a base model with a raw completion prompt, so this never happened to the authors. Repairing the answer prompt moved the ceiling from 11.6 to 51.4 percent on exact match while leaving the lenient grader almost unchanged, which proves the fix removed padding words rather than adding correctness. Roughly forty points of apparent failure were pure formatting.

2. **The lenient grader was suspected of rewarding regurgitation, and the suspicion was tested and rejected.** After finetuning, the model recites passage text instead of answering, and the gold answer is a span inside the passage by construction, so a substring grader could hand out credit for nothing. A shuffled gold control, which grades every prediction against a different question's answer, put that inflation at 0.1 points, or 0.5 percent of the measured gain. The worry was real as a mechanism and small as a number. It is reported either way.

**Baseline A, finetuning on the raw passage, reproduces the paper's claim in substance.**

|  | base | + raw passage | gain | paper at 7B |
|---|---|---|---|---|
| Exact match | 0.7% | 0.7% | **0.0** | — |
| Contains | 3.6% | 31.9% | +28.3 | 32.7 to 33.5, **+0.8** |
| F1 >= 0.6 | 2.9% | 2.9% | **0.0** | — |

The graders disagree completely and the disagreement is the result. After memorising the passage the model answers by reciting nearby passage text. Recitation contains the answer but recitation is not answering. Exact match demands the model isolate the span and it gains **exactly zero points**. So the paper's claim that raw passage text is a poor teacher holds here with unusual clarity, even though the lenient number looks large.

## Status

In progress. Built and verified so far: the measurement and its ceiling, floor and false positive rate, LoRA, the inner loop, and Baseline A. Still to come: the self-edit generator, the ReST-EM outer loop, accuracy across RL rounds, the catastrophic forgetting experiment, and the prompt ablation.

The notebook is added manually.
