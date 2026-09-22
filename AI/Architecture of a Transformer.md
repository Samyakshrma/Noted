

**A transformer doesn't transform the sentence. It repeatedly edits one vector per token, and attention is the only place those vectors are allowed to talk to each other.**

Almost every confusing thing about transformers dissolves once that's your mental picture instead of "data flows through layers."

## The shape of the whole thing

The job is: given some tokens, produce a probability distribution over what token comes next. Everything inside serves that.Let me walk through each stage, then park on the middle one for a long time.

## Getting in: tokens, embeddings, position

Text gets chopped into **tokens**, which are subword chunks rather than words. `"unhappiness"` might become `un` + `happiness`. Each token is just an integer ID, an index into a vocabulary of maybe 50,000 entries.

Then comes the embedding matrix, `W_E`, of shape `[50000, 768]`. Token ID 1834 means "take row 1834." That's the whole operation. No matrix multiply, just a lookup.

So `"the cat sat"` becomes three vectors of 768 numbers each. These 768 numbers are learned during training, and training arranges them so that tokens used in similar ways end up pointing in similar directions.

There's a problem here, and it's important: attention has no idea what order things are in. The math is symmetric across positions. Without help, "dog bites man" and "man bites dog" produce identical outputs. So position gets injected, either by adding a position-dependent vector to each embedding, or (in modern models) by RoPE, which rotates each token's query and key vectors by an angle proportional to its position. The rotation trick is elegant because when you dot two rotated vectors together, the result depends only on the _difference_ in their angles, so the model naturally sees relative distance rather than absolute index.

## The residual stream: the idea everything hangs on

Now each token has its vector. Here's the part most explanations skip, and it's the thing that makes the rest obvious.

That vector is not consumed and replaced by each layer. It _persists_, all the way from the embedding to the final output, and every component **adds** into it:

```
x ← x + attention(x)
x ← x + mlp(x)
```

This running vector is called the **residual stream**.Four consequences fall out of this, and they're worth sitting with:

**It's a shared workspace, not a pipeline.** Layer 30 can read something layer 2 wrote, directly, without layers 3 through 29 having to forward it. Information written early stays available.

**Nothing is destroyed by default.** A component with nothing useful to contribute can output near-zero, and the stream passes through untouched. Compare that to a design where each layer replaces the vector: then every layer is forced to reconstruct everything it wants to keep.

**The 768 dimensions act as channels.** Different directions in that space get used for different purposes. Some subspace might carry "this is a plural noun," another "the current sentence is a question," another "we're inside a code block." A component reads the directions it cares about and writes to the directions it wants to broadcast on. They coexist because the space is high-dimensional enough that many near-orthogonal directions fit.

**Gradients have a free path.** Because addition passes gradient through unchanged, there's a straight highway from the loss back to the embeddings. This is the reason you can stack 96 of these and still train it.

One small but real detail: each component reads a **LayerNormed copy** of the stream, not the stream itself. The stream keeps accumulating, so its magnitude grows; LayerNorm hands each sub-layer a cleanly rescaled view (subtract the mean, divide by the standard deviation, then apply a learned scale and shift) so the sub-layer doesn't have to cope with wildly varying input sizes. Modern models often use RMSNorm, which skips the mean subtraction and just divides by the root-mean-square. The stream's own magnitude is never touched.

## Attention, properly

Attention is the only operation in the entire architecture where information crosses between token positions. Everything else is strictly per-token. So every time a model does something that requires combining words, it happened here.

### Three views of the same vector

Each token's vector gets projected three ways by three learned matrices:

|Projection|Formula|What it represents|
|---|---|---|
|Query|`q = x · W_Q`|what this token is looking for|
|Key|`k = x · W_K`|what this token advertises about itself|
|Value|`v = x · W_V`|what this token hands over if selected|

This is the conceptual heart of attention, so let me make it concrete. Think of a room of people. Your **query** is the question you shout: "who here is a singular animate noun?" Everyone's **key** is the badge on their chest: "I'm a preposition," "I'm a proper noun," "I'm a number." Your query gets compared against every badge. Whoever matches best, you walk over to them, and what you actually take away is their **value**, which is a different thing from their badge. The badge is for being found; the value is the goods.

Q, K, and V being separate matrices is what makes this work. A token's criteria for being _retrieved_ can be completely different from the information it _provides_ when retrieved. The word "cat" might advertise "I'm a nominative singular noun" in its key while its value carries "feline, animal, small, domestic."

### The matching

The score between token `i` looking and token `j` being looked at is a dot product:

```
score[i][j] = q_i · k_j / √d_head
```

Dot product measures alignment. If `q_i` and `k_j` point the same way, the score is large. That's all "relevance" means mechanically.

The `√d_head` division isn't cosmetic. The dot product of two `d`-dimensional vectors accumulates `d` terms, so its variance scales with `d`. At `d = 64`, raw scores drift into the tens, softmax saturates into a near one-hot spike, and the gradient through it dies. Dividing by `√d` keeps the scores at roughly unit variance so softmax stays soft and trainable.

Then the **causal mask**: every `score[i][j]` where `j > i` is set to negative infinity. Position 5 cannot see position 6. This is what makes the model a valid next-token predictor, and it's also what lets you train on every position of a sequence simultaneously instead of one at a time.

Then softmax across each row, turning scores into weights that sum to 1. Then the output is a weighted average of the value vectors:

```
z_i = Σ_j attention[i][j] · v_j
```

Here's what one head of that looks like:The numbers on the boxes are the softmax weights, and they sum to 1. The lower fan is the query matching against keys; the upper fan is the values getting pulled back in proportion. Notice the output is literally a blend, so whatever "cat" carries in its value vector now lands in "sat"'s residual stream.

A concrete example that shows why this is powerful: in _"The cat that the dog chased was tired,"_ when the model processes "was," it needs to know the subject is singular. A head whose query encodes "find the head noun of my clause" will match "cat" and not "dog," even though "dog" is closer. Attention has no notion of distance unless the model chooses to build one. Any two tokens are one hop apart.

### Multi-head

One attention pattern gives one weighted average, which means one lookup per position. But a token typically needs several things at once: its grammatical subject, the verb it modifies, the entity a pronoun refers to, whether this phrase appeared earlier in the document.

So the model runs several attention operations in parallel. With `d_model = 768` and 12 heads, each head gets `d_head = 64`. Each head has its own `W_Q`, `W_K`, `W_V`, each of shape `[768, 64]`. Every head sees the full residual stream but projects it down into its own 64-dimensional subspace, where it can pay attention to its own thing without interference.

Heads do not talk to each other inside the layer. They run completely independently, then their 12 outputs of 64 dims are concatenated back to 768 and passed through an output matrix `W_O` of shape `[768, 768]` before being added to the stream. `W_O` is what lets each head choose _which directions of the residual stream to write into_.

Here's the full shape trace for one layer with 5 tokens:

|Step|Shape|
|---|---|
|Input from stream|`[5, 768]`|
|Q, K, V per head|`[5, 64]` each, ×12 heads|
|Scores `QKᵀ`|`[5, 5]` per head|
|After mask + softmax|`[5, 5]`, rows sum to 1|
|Weighted values|`[5, 64]` per head|
|Concatenated|`[5, 768]`|
|After `W_O`|`[5, 768]`, added to stream|

That `[5, 5]` matrix is the whole reason context length is expensive. Double the sequence and it quadruples.

## The MLP: where the thinking happens

After attention writes its result, the block does it again with a completely different kind of component.

The MLP (also called the feed-forward network) is applied to **each position independently and identically**. It sees no other token. If attention is the step where a token gathers what it needs, the MLP is the step where it sits alone and thinks about what it just gathered.

Structurally it's tiny:

```
h = activation(x · W_in)     # 768 → 3072
out = h · W_out              # 3072 → 768
```

The interesting reading is as a **key-value memory**. Each of the 3072 rows of `W_in` is a detector direction. The dot product `x · w_k` is large when the residual stream contains that feature. The activation function gates it: strong match passes through, weak match gets suppressed toward zero. Then column `k` of `W_out` is that neuron's stored output vector, the thing it writes into the stream when it fires.Every neuron inspects the vector; only a sparse subset fires meaningfully; the output is the sum of what those firing neurons want to say. I should be honest that the clean one-concept-per-neuron labels are a simplification — real neurons are usually polysemantic, firing for several unrelated things, because the model packs far more features than it has dimensions by using directions that are merely _almost_ orthogonal. But the mechanism is exactly as pictured.

Two "why" questions that matter here:

**Why expand to 4×?** More room for features than the residual stream has dimensions. The stream is a communication bus and stays narrow; the MLP's hidden layer is scratch space and goes wide.

**Why a nonlinearity at all?** Without it, `W_in` and `W_out` collapse into a single matrix and the whole MLP becomes one linear map, which attention could already do. The nonlinearity is what lets the model compute conditional logic: fire _only if_ this feature is present _and_ that one is too.

Worth knowing: the MLPs hold roughly two-thirds of a transformer's parameters. A lot of what looks like factual knowledge lives in them.

## The block, assembled

Now put both halves together. This is the unit that repeats.The rhythm is: **gather, then think, gather, then think**, with everything landing back in the same accumulating vector.

Two notes on this layout. First, the layer norms sit _before_ each sub-layer rather than after. This is "pre-norm," and it's the modern standard because it keeps the residual path completely clean (pure addition all the way down), which makes deep stacks trainable without careful warmup. The original 2017 paper put the norms after the additions and was noticeably fussier to train.

Second, the attention and MLP are doing genuinely complementary work. Attention is a routing operation and its weights depend on the data (which token attends where changes per input), but given those weights it's just a weighted average — mild computation. The MLP does no routing at all but does heavy nonlinear computation. Neither one alone is enough. An attention-only transformer struggles to compute much; an MLP-only one is a per-token map with no context.

## What changes as you stack them

Stack 12, or 32, or 96 of these. Same structure each time, different weights. A loose but real trend in what they specialize in:

- **Early blocks** do detokenization and local syntax. They merge subword fragments into coherent concepts and resolve very local structure. A lot of layer 0 and 1 work is undoing the damage the tokenizer did.
- **Middle blocks** do the semantic heavy lifting: tracking entities, resolving what pronouns point at, retrieving facts, following syntactic structure across distance.
- **Late blocks** converge toward the actual prediction, moving the answer into the position where it can be read out.

One of the most beautifully documented mechanisms across layers is the **induction head**, which is a two-layer circuit. In one layer, a head copies each token's identity into the next position. In a later layer, a head uses that to search backward for "where did this current token appear before?" and then copies whatever followed it. That's the machinery behind a model completing a name, a repeated phrase, or a pattern you established three sentences ago. No one designed it; it emerges during training.

## Getting out

After the last block, one final layer norm, then the unembedding: multiply the 768-dim vector by `W_U` of shape `[768, 50000]` to get 50,000 logits. Many models tie this to the transpose of the embedding matrix, reusing the same weights.

This gives you the cleanest possible statement of what the network is actually for. The logit for token `t` is the dot product of the final residual vector with token `t`'s direction. So the entire job of every block, every head, every neuron, is:

**steer the residual vector to point toward the correct next token.**

Softmax turns those logits into probabilities. Then sampling (temperature, top-k, top-p) picks one, it gets appended to the sequence, and the whole thing runs again.

## Two practical things that make it all make sense

**Training is embarrassingly parallel.** Because of the causal mask, you can feed in a 2000-token sequence and get predictions at all 2000 positions in one forward pass. Position 100 predicts token 101 while position 700 predicts token 701, all simultaneously, and none of them can cheat by looking ahead. Cross-entropy loss is averaged across all of them. An RNN has to go one step at a time; this is why transformers won.

**Generation reuses almost everything.** When generating token by token, the keys and values of past tokens never change, since the causal mask means they were never influenced by anything that came later. So you cache them. Each new token computes only its own `q`, `k`, `v` and attends over the stored cache. Without it, generating 1000 tokens means 1000 full forward passes over the growing prefix.

## The compressed version

If you keep only one paragraph, keep this one. Each token carries a vector. That vector persists through the entire network and every component only ever adds to it. Attention is the one place tokens exchange information: each token issues a query, matches it against every other token's key, and pulls back a weighted blend of their values. The MLP is where each token, alone, runs what it gathered through thousands of pattern detectors and writes back whatever fired. Do those two things alternately, dozens of times, and by the end the vector points at the right next word.

Want me to go deeper on any one piece? The places worth another pass are usually the QKV intuition with actual small numbers you can follow by hand, or the induction head circuit in detail. I can also turn this into a page you can keep and come back to.