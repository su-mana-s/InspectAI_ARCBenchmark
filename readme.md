## The What
Evaluations on The ARC Challenge Benchmark With The Inspect Library
- If the nb does not render, see - [nbviewer](https://nbviewer.org/github/su-mana-s/InspectAI_ARCBenchmark/blob/main/AlgoVerseBenchmarkingChallenge.ipynb)
- The project analyses the performance of GPT models on the ARC challenge bench, constructing an Inspect eval pipeline, complete with custom solvers and scorers for multiple eval paradigms.
- Logprobs extraction from model responses to calculate confidence scores and further logprobs-based scoring are implemented

----------
## Overview
### Part A
- Setup - Normal benchmark - test/validation sets - Samples - inbuilt solvers and scorers
### Part B
- EvalSet - 2 models - Comparison - Analysis of Logs, Results 
### Part C
- Currently, the inspect ai library does not support returning logprobs directly from model calls.
- LogProbs integration - custom solvers and scorers for different metrics(accuracy, stderr, mean over answers, confidence, topk etc), plots
### Part D
- If you allow for multiple answers, the models usually end up choosing more than is necessary. Even though they get the right answer in a single answer setting.
1. Top logprobs for multi answers - are the 'just right' answers in top k?
2. Will letting the model know that there will be negative marking, ie., penalty by avg help?


## Summary of Analysis/Insights

### A. Basic
1. Model: gpt-4.1-nano.
2. Data: 250 samples from the validation set of the ARC challenge dataset.
3. Accuracy: 0.892, stderr: 0.020
4. solver, scorer: multiple_choice, choice (inbuilt)

### B. Moderate
1. Accuracy- on limit=250 samples from the validation set
    - gpt-4o-mini: 0.916
    - gpt-4.1-nano: 0.888
2. 
    - Questions that Model 1(gpt-4.1-nano) got right that Model 2(gpt-4o-mini) didn't: 10
    - Questions that Model 2(gpt-4o-mini) got right that Model 1(gpt-4.1-nano) didn't: 19
3. There are 14 questions that both models get wrong having chosen the same wrong options, and 3 questions that they get wrong choosing different wrong options.

    - Of these, atleast 1 had different labels ie., [1,2,3,4] instead of [A,B,C,D]. Since the model was prompted to respond with one of the labels it responded with '4' where the target was 'D'. Essentially, it got the answer right, but the question formatting was wrong. (Deliberate on the authors' part, I wonder?)

### C. Advanced

1. Set-up: Custom Scorers
    - Accuracy, stderr: Top K Winner, k=2 ie., a larger margin for error
    (Say you want to give it an easier chance of winning, so maybe take k=2
    ie., if the correct ans (target) is in the model's first 2 choices
    here, greedy pick is essentially k=1)
    - Accuracy, stderr: Greedy Pick - ie., acc and stderr of the ans its most confdent in
    - Mean score: Average % confidence over all records - by linear probability - for the target
    - Mean score: Average % confidence of greedy ans over all records - by linear probability

2. The greedy score, ie., the top choice with k=1 align with the initial comparisons above for both models.
3. The topk_winner scores, on the other hand, with just k=2, sees mini score a full accuracy = 1 and nano a close second with accuracy = 0.996 !
(ie., given just a very minute increase in margin for error - a second chance, essentially - the models generally get everything right?!)
4. Confidence is calculated as the linear probability from the logprobs of top-k answers. Top-conf., ie., the model's confidence in its top choice is at a 99% for mini and 97% for nano! 
5. Whereas the confidence in the actual, ground-truth target answer is a 90% and 87% for mini and nano, respectively. This is because, well, they are generally confident of their top choices, regardless of whether they are right or wrong. So, when they get some questions wrong and the target differs, their confidence in their wrong answer remains - thus the lower scores.

### D. MC-MCQ - Multi-correct Answers

- What happens when models are told that there might be more than 1 answer to a question? A more realistic scenario might involve MCMCQs!
- Granted, the data being used doesnt have multi-correct answers-  which makes it even more concerning that models score abysmally(worse than normal) by choosing more options than are necessary.

- Set-up: Inspect's Multiple choice prompt for the user prompt. 
1. Without logprobs - Custom scorer: partial_choice which affords partial credit by averaging correct keys from the set of all returned keys in a multi-choice-multi-correct scenario. ie., in a multi-correct scenario, if the model had returned the perfect target it would have achieved a score of 1. Adding, say, just one other useless option brings the score down to 0.5.
    - Both models tend to over-answer 
    - gpt-4.1-nano - 0.803
    - gpt-4o-mini - 0.751
But mini was performing better all this while! The scores are significantly worse than from the single answer prompts above!

2. With logprobs- Custom scorers: A) Our penalising MCMCQ scorer and B) mc_topk_winner that calculates the accuracy by checking for the correct answer set in top k (here, k=2). Note that this requires an exact match.
So, if our required target isnt in top-k, the scorer doesnt even give partial credit- it gives 0.
Under this, we have 2 paradigms - w/ penalty and w/o penalty., ie., do we let the model know that it is being penalised?

3. w/o penalty - ie., normal prompt for MCMCQ, the model doesn't know that its being penalised for adding wrong answers.
    - Analysing the same with logprobs - ie., given that it returns the wrong answers, is the correct answer in topk? The scores below are for topk=2
    - mini - 0.816 (mc_topk_winner)
    - nano - 0.68 (mc_topk_winner)
    - The scores might be lower than the original MCMCQ scores - but these are not meant to be compared! Note definition  of B above.
    - See snapshots down below

4. w/ penalty - ie.,  What if you tell the model that there's a penalty for extra answers? 
    - Funnily enough - for some cases, the correct answer manages to make it to atleast one of topk=4, whereas in the previous case, it was nowhere to be found. The scores below are for topk=2
    - mini - 0.828 (mc_topk_winner)
    - nano - 0.756 (mc_topk_winner)
    - Slight improvement?



