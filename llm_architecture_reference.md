# LLM Architecture — A Personal Reference

> A working understanding of how a transformer-based LLM (Llama 3 style) processes text end-to-end. Built around the running example: predict the next token after "The cat sat on the ___". Concept first, mechanism second, mathematical detail only where it earns its place.

---

## Table of contents

1. [The 30-second mental model](#1-the-30-second-mental-model)
2. [Stage 1 — Tokenization](#2-stage-1--tokenization)
3. [Stage 2 — Embeddings](#3-stage-2--embeddings)
4. [Stage 2b — Positional Encoding (RoPE deep dive)](#4-stage-2b--positional-encoding-rope-deep-dive)
5. [Stage 3 — Attention (Q, K, V)](#5-stage-3--attention-q-k-v)
6. [Stage 4 — Feed-Forward (FFN)](#6-stage-4--feed-forward-ffn)
7. [Stage 5 — Output Head](#7-stage-5--output-head)
8. [The complete pipeline](#8-the-complete-pipeline)
9. [Parameter inventory — where the 8B numbers live](#9-parameter-inventory--where-the-8b-numbers-live)
10. [KV Cache — deep dive](#10-kv-cache--deep-dive)
11. [Glossary & vocabulary](#11-glossary--vocabulary)
12. [Cheat sheet](#12-cheat-sheet)
13. [Personal Q&A log](#13-personal-qa-log)

---

## 1. The 30-second mental model

An LLM is a machine that does this:

```
Text → Numbers → Numbers that "understand context" → Numbers → Text
```

Five stages:

1. **Tokenization** — chop text into pieces, assign each an ID
2. **Embeddings (+ position)** — turn each ID into a meaning-carrying vector
3. **Attention** — let each token "look at" the others to gather context
4. **Feed-forward** — let each token "think" about what it learned
5. **Output head** — turn the final vector back into a predicted token

Stages 3 and 4 form one **layer**, and the model stacks many of them (Llama 3 8B = 32 layers).

Running example throughout: predict what comes after **"The cat sat on the ___"**.

---

## 2. Stage 1 — Tokenization

### What

Split text into sub-word tokens, assign each a numeric ID from a fixed vocabulary.

```
"The cat sat on the"
       ↓
["The", " cat", " sat", " on", " the"]
       ↓
[464, 3797, 3332, 319, 262]
```

### Key points

- Tokens are usually **sub-word** pieces, not whole words. This handles unknown words by composing them from known pieces and keeps vocab size manageable (~128K for Llama 3, ~50K for GPT-2).
- Leading spaces are **part of the token** — ` cat` and `cat` are different tokens.
- Capitalization matters — `The` and ` the` are different tokens.
- Algorithm used is typically **BPE (Byte-Pair Encoding)** or a variant — learned from training data.

### Practical implications

- A model has a token budget (context length), not a word budget.
- Rough rule: 1 token ≈ 0.75 English words. Code, JSON, and non-English text are much less efficient (often 1 token ≈ 1 character).
- A Parquet schema dumped as JSON may consume 3× the tokens of the same info written as prose.

---

## 3. Stage 2 — Embeddings

### What

Each token ID looks up a row in a large **embedding table** to retrieve a learned vector that represents that token's "meaning."

```
Token ID 3797 (" cat")  →  embedding vector [0.91, 0.22, -0.15, ...]   (4096 numbers)
```

For Llama 3 8B, every token becomes a vector of **4096 numbers**. The embedding table is shape `[vocab_size × embedding_dim]` = `128,256 × 4096` ≈ 525 million numbers — all learned.

### Mental model

> Imagine a 4096-dimensional space where every possible token has its own location. Words with similar meanings end up near each other. "cat" near "dog" / "kitten"; "sat" near "stood" / "rested".

You can't visualize 4096 dimensions. Nobody can. Trust the math.

### What's missing at this point

The token vector knows **what** it is, but not **where** it is in the sequence. Without position info, `[The, cat, sat, on, the]` and `[the, on, sat, cat, The]` would look the same to the model. We fix this next with positional encoding.

---

## 4. Stage 2b — Positional Encoding (RoPE deep dive)

### What positional encoding is (and isn't)

Positional encoding adds **position information** to the embedding. Crucially, it's not an algorithm that "looks at" the input intelligently — it's a deterministic stamp based purely on position number.

> The embedding is the guest (who they are). The position encoding is the room number stamp (where they are). Room 3 always gets the same stamp regardless of who's staying there.

### Two main flavors

| Flavor | Used in | Mechanism | Where applied |
|---|---|---|---|
| **Sinusoidal / absolute** | Original Transformer (2017) | Fixed sine/cosine patterns per position, **added** to embedding | Once, at input |
| **RoPE (Rotary)** | Llama, GPT-NeoX, most modern models | **Rotates** the vector by an angle proportional to position | Inside every attention layer, on Q and K |

### RoPE — the intuition

Rotate each token's vector by an angle proportional to its position. Tiny toy example (2D instead of 4096D, 30° per position):

```
"cat" embedding = [1.0, 0.0]   (points right →)

Position 1:  rotated by 30°  → [0.87, 0.50]
Position 2:  rotated by 60°  → [0.50, 0.87]
Position 3:  rotated by 90°  → [0.00, 1.00]   (points up ↑)
```

Same word, same starting vector, different final direction based on position.

### The killer property — relative position falls out for free

When attention later compares two tokens via dot product, the math works out so **only the *difference* in their rotations matters**.

- "cat" at position 2 → rotated 60°
- "mat" at position 5 → rotated 150°
- Their similarity score depends on `150° − 60° = 90°` → corresponds to a relative distance of 3 positions.

The model doesn't care about absolute positions — only relative gaps. Shift the whole sentence right by 10 positions and the similarity scores between any two tokens stay identical.

### Why this is a feature, not a bug

Two tokens' similarity score now depends on:
1. **Content** — what they are semantically
2. **Position gap** — how far apart they are

This is *desirable*. In language, the relationship between two words depends heavily on their distance:

> "The cat sat on the **mat**" — "cat" and "mat" are 4 tokens apart, tightly related.
> "The cat sat on the mat. Yesterday I went to the **mat** store." — same word "mat", now far away, much weaker relationship.

RoPE bakes this into the geometry directly.

### Scaling to real dimensions

In a 4096-dim vector, RoPE doesn't rotate the whole thing as one big arrow. It chops the vector into **2048 pairs of dimensions** and rotates each pair independently, each at its own frequency. Some pairs rotate fast (sensitive to nearby positions), some slow (sensitive to long-range positions). Multi-scale position awareness, like a clock with hour/minute/second hands.

### Nothing gets "reset"

> Once rotated, vectors stay rotated. They are not normalized back for comparison. The dot product math naturally makes only the *relative* angle matter when comparing two rotated vectors.

### RoPE and context length

- RoPE compute itself is cheap — not a bottleneck.
- **The real long-context limit:** the model only saw certain rotation angle ranges during training. Beyond that, behavior degrades because the angles are out-of-distribution.
- Techniques like **YaRN**, **NTK-aware scaling**, **position interpolation** address this — they rescale RoPE frequencies so a model trained at 4K can extend to 128K.

---

## 5. Stage 3 — Attention (Q, K, V)

This is the heart of the transformer.

### The problem attention solves

After embeddings + position, each token's vector knows what it is and where it is, but knows nothing about the surrounding tokens. Attention rewrites each vector to include context.

### The dinner-party analogy

For each token, three things happen:
1. **Query (Q)** — the token forms a silent question: "what context do I need?"
2. **Key (K)** — every token (including itself) broadcasts an advertisement: "here's what I offer"
3. **Value (V)** — every token also prepares actual content: "here's what you get if you listen to me"

The token's query is compared against every key to score matches. The best-matching keys' values get blended in.

### Step-by-step on our example

Focus on " the" (the last token, which is trying to predict the next word).

**1. Form Q, K, V via learned projections.** Three matrices `W_Q`, `W_K`, `W_V` (each ~4096×4096) project the input vector into role-specific versions:

```
v_the  → W_Q → q_the     (its query)
v_i    → W_K → k_i       (each token's key)
v_i    → W_V → v_i_payload (each token's value)
```

**2. Score every pair via dot product.** Dot product = how aligned two vectors are.

```
q_the · k_The  = 0.16
q_the · k_cat  = 0.33
q_the · k_sat  = 0.84
q_the · k_on   = 1.19    ← highest
q_the · k_the  = 0.16
```

` on` wins because q_the and k_on point in similar directions in vector space (large components in the same dimensions). The W_Q and W_K matrices, trained over billions of examples, **learned** to project tokens such that productive matches (determiner-after-preposition needs the preposition's context) land in geometrically aligned positions.

**3. Softmax → attention weights** (probabilities summing to 1):

```
The   → 0.05
cat   → 0.13
sat   → 0.30
on    → 0.42    ← dominates
the   → 0.10
```

**4. Weighted sum of values:**

```
new_v_the = 0.05·v_The_payload
          + 0.13·v_cat_payload
          + 0.30·v_sat_payload
          + 0.42·v_on_payload
          + 0.10·v_the_payload
```

Two operations: scalar × vector (each weight scales its V), then vector addition. Result is a single new vector of the same size — the new representation of " the", now flavored heavily by " on".

### Important: this happens in parallel for every token

Each token forms its own query and computes its own weighted sum. All five updates happen simultaneously on GPU:

```
v_The_new = weighted sum based on q_The · all keys
v_cat_new = weighted sum based on q_cat · all keys
v_sat_new = weighted sum based on q_sat · all keys
v_on_new  = weighted sum based on q_on  · all keys
v_the_new = weighted sum based on q_the · all keys
```

### Causal masking (decoder models)

When computing attention for token at position `i`, it can only attend to positions `1..i`, not future ones. Without this, training next-token prediction would let the model cheat by peeking.

```
v_The_new : attends to {The}                      (1 token)
v_cat_new : attends to {The, cat}                 (2 tokens)
v_sat_new : attends to {The, cat, sat}            (3 tokens)
v_on_new  : attends to {The, cat, sat, on}        (4 tokens)
v_the_new : attends to {The, cat, sat, on, the}   (5 tokens)
```

### Multi-head attention

One attention computation isn't enough. The model does it in parallel `H` times with different W_Q, W_K, W_V matrices each — each parallel run is an **attention head**. Llama 3 8B has 32 heads per layer. Different heads learn to attend to different patterns (syntax, coreference, long-range dependencies, etc.).

### The W_Q / W_K / W_V matrices — what they are

- Each is a learned matrix of numbers (~16.8M numbers each in Llama 3 8B).
- They are projections — same input vector, three different "lenses" extracting three role-specific views (question / advertisement / payload).
- Learned through training. Initialized randomly; gradient descent shapes them over billions of examples.
- **At inference, frozen.** Same matrices applied to every input.
- Context-dependence comes from the **inputs** to these matrices, not the matrices themselves. Same W_Q applied to "sat" in different sentences produces different queries because the input `v_sat` already reflects different surrounding context (mixed in by earlier layers).

> The Ws are fixed lenses. The token vectors are the slides under the lens. The lens doesn't change — the image does, because the slide changes.

### Residual connection

In practice, the attention output is **added back** to the original input: `output = input + attention(input)`. This **residual / skip connection** lets information flow through layers without being totally rewritten, and makes deep stacking trainable.

---

## 6. Stage 4 — Feed-Forward (FFN)

### What it does

Attention mixes information **between** tokens. Feed-forward **processes** the information at each token, independently.

> Attention = the meeting where everyone shares info. FFN = each person returning to their desk to digest what they learned.

### Mechanics — expand, activate, squeeze

For each token, independently:

```
v  →  W_up  →  expand to ~4× size  →  apply non-linearity  →  W_down  →  back to original size
```

For Llama 3 8B:
- Input: 4096-dim
- Expanded: 14,336-dim (≈ 3.5× — varies by model)
- Output: 4096-dim

### Why expand-then-squeeze

The expansion gives the model **room to compute**. The non-linearity lets it select features (amplify some, suppress others). The squeeze produces a same-size output the next layer can use.

Without the non-linearity, two matrix multiplications would collapse mathematically into one bigger matrix — useless. The activation function is what gives the FFN real computational power.

### Modern variant: SwiGLU

Llama-style models don't use the simple "expand → activate → squeeze" pattern. They use **SwiGLU**, which adds a third matrix `W_gate`:

```
expanded_a = input · W_up
expanded_b = input · W_gate       (in parallel)
gated      = expanded_a * activation(expanded_b)   (element-wise product)
output     = gated · W_down
```

`W_gate` acts as a learned gate selecting which features of the expanded representation pass through. Empirically works better than vanilla expand-squeeze.

### What FFN "does" intuitively

Recent interpretability research suggests FFN layers act like a **key-value memory** or lookup table. Specific hidden units in the expanded space fire on specific patterns ("this is a past-tense verb", "this is about cooking") and add corresponding features back to the output. Many factual associations (Paris → France) seem to live in FFN layers rather than attention.

### Per-token, independent

FFN has no cross-token communication. Each token's vector is processed in isolation. This is fine because attention already did the mixing.

### Why FFN dominates parameter count

```
Per Llama 3 8B layer:
  Attention (Q/K/V/O):  ~42 M params
  FFN (gate/up/down):  ~176 M params
                       ─────────
                       ~218 M params per layer
```

Roughly **2/3 of model parameters live in FFN.** Attention dominates *compute cost* (N² scaling); FFN dominates *parameter count*.

---

## 7. Stage 5 — Output Head

### What

After 32 layers, the final vector for the last token contains the model's best summary of "what should come next." The output head converts this into an actual token.

### Three sub-steps

**1. Multiply by the LM head matrix** (shape 4096 × 128,256):

```
final_vector (4096) × W_LM_head → logits (128,256 numbers, one per vocabulary token)
```

Each column of W_LM_head is essentially "the signature of one specific token." The matrix multiplication is a bulk dot-product comparison: how similar is the final vector to every possible token? Tokens whose signatures align well get high logits.

> This is your earlier intuition formalized: prediction = nearest token in vector space. Not just one nearest — a full ranked similarity score against every token.

**2. Softmax → probabilities** (summing to 1):

```
" mat"   →  32%
" floor" →  18%
" couch" →  11%
" rug"   →   9%
" bed"   →   7%
...
```

**3. Sample a token:**

| Strategy | Behavior |
|---|---|
| **Greedy** | Always pick the highest-probability token. Deterministic. |
| **Temperature** | Sample with weights from the softmax. Temperature < 1 sharpens (more greedy); > 1 flattens (more random); = 0 is greedy. |
| **Top-k** | Keep only top-k candidates, then sample. |
| **Top-p (nucleus)** | Keep the smallest set of tokens whose probabilities sum to ≥ p, then sample. |

This is exactly where Ollama's `temperature`, `top_k`, `top_p` parameters plug in.

### Autoregressive generation

Append the predicted token to the sequence and run the whole forward pass again to predict the next one. Repeat until end-of-sequence or max length:

```
"The cat sat on the"  →  " mat"
"The cat sat on the mat"  →  "."
"The cat sat on the mat."  →  " It"
...
```

---

## 8. The complete pipeline

```
"The cat sat on the"
        │
        ▼
   TOKENIZATION                      [464, 3797, 3332, 319, 262]
        │
        ▼
   EMBEDDING LOOKUP                  5 vectors of size 4096
        │
        ▼
   ┌────────────────────────────────────┐
   │  LAYER 1                           │
   │   ├─ RMSNorm                       │
   │   ├─ Attention (Q, K, V + RoPE)    │ ← tokens look at each other
   │   ├─ + Residual                    │
   │   ├─ RMSNorm                       │
   │   ├─ FFN (gate, up, down — SwiGLU) │ ← each token processes alone
   │   └─ + Residual                    │
   ├────────────────────────────────────┤
   │  LAYER 2 — same structure          │
   ├────────────────────────────────────┤
   │  ...                               │
   │  LAYER 32                          │
   └────────────────────────────────────┘
        │
        ▼
   Final RMSNorm
        │
        ▼
   LM HEAD (4096 × 128,256)          Multiply final vector by W_LM_head
        │
        ▼
   Logits (128,256 numbers)
        │
        ▼
   Softmax → probabilities
        │
        ▼
   Sample (greedy / temperature / top-p)
        │
        ▼
   " mat"
```

Early layers tend to handle surface features (syntax, local patterns). Middle layers handle structural relationships. Late layers handle semantic abstractions.

---

## 9. Parameter inventory — where the 8B numbers live

A "parameter" is one single learned number inside any weight matrix. "8B parameters" = 8 billion such numbers.

### Full Llama 3 8B inventory

| Component | Shape | Parameters |
|---|---|---|
| Token embedding table | 128,256 × 4096 | ~525 M |
| LM head | 4096 × 128,256 | ~525 M |
| **Per layer (× 32 layers):** | | |
| &nbsp;&nbsp;W_Q | 4096 × 4096 | 16.8 M |
| &nbsp;&nbsp;W_K | 4096 × 1024 (GQA) | 4.2 M |
| &nbsp;&nbsp;W_V | 4096 × 1024 (GQA) | 4.2 M |
| &nbsp;&nbsp;W_O (attention output) | 4096 × 4096 | 16.8 M |
| &nbsp;&nbsp;W_gate (FFN) | 4096 × 14,336 | 58.7 M |
| &nbsp;&nbsp;W_up (FFN) | 4096 × 14,336 | 58.7 M |
| &nbsp;&nbsp;W_down (FFN) | 14,336 × 4096 | 58.7 M |
| &nbsp;&nbsp;RMSNorm × 2 | 4096 each | ~8 K |
| **Per-layer total** | | ~218 M |
| **32 layers** | | ~7.0 B |
| **Grand total** | | **~8.05 B** |

### Where the parameters go, proportionally

- **~13%** — embeddings + LM head ("vocabulary lookup" layers)
- **~17%** — attention across all layers
- **~70%** — feed-forward across all layers

### Grouped-Query Attention (GQA) — why K and V are smaller

In Llama 3, multiple query heads share one K/V pair. 32 Q heads + 8 KV heads. Cuts KV cache size 4× with minimal quality loss.

### What "the model" actually is

When you download Llama 3 from Hugging Face, you download gigabytes of files containing the numbers that fill all these matrices, plus a tiny bit of architecture code describing how to wire them together. **The trained model = its weights.**

Fine-tuning = nudging some or all of these numbers further via additional training. Architecture stays identical.

---

## 10. KV Cache — deep dive

### Why it exists

Without caching, generating each token would require re-running the entire model on the entire growing sequence. Painfully slow — O(N²) work to generate N tokens.

### The architectural property that enables caching

Once a token's K and V have been computed in a given layer, they **will never change**, no matter how many more tokens are generated afterward. Why? Because of causal masking — future tokens never affect past tokens. So K and V for past tokens are immortal once computed. Cache them.

### Two phases of inference

| Phase | What happens | Bottleneck |
|---|---|---|
| **Prefill** | Process the whole prompt in one big parallel forward pass. Compute Q/K/V for every token, every layer. Populate KV cache. | Compute (FLOPs) |
| **Decode** | Generate one new token at a time. Compute Q/K/V for just the new token. Append new K, V to cache. Run attention against full cached K, V. | Memory bandwidth |

Counterintuitive consequence: a 1000-token prompt takes much less wall time than generating 1000 tokens. Prefill is parallel; decode is sequential.

### Cache size math

Per token, per layer, you cache K and V:

```
Llama 3 8B:
  num_kv_heads × head_dim = 8 × 128 = 1024 numbers each for K and V
  Per layer per token: 2 × 1024 = 2048 numbers
  All 32 layers: 32 × 2048 = 65,536 numbers per token
  At fp16: ~131 KB per token
```

| Context | KV cache size (Llama 3 8B, fp16) |
|---|---|
| 1K tokens | 128 MB |
| 4K tokens | 512 MB |
| 8K tokens | 1.0 GB |
| 32K tokens | 4.2 GB |
| 128K tokens | 16.8 GB |

For comparison, the 8B weights at fp16 = ~16 GB. **At 128K context, the cache is as large as the model itself.**

### The real limiters of context length

1. **Attention's N² compute** — every token compares to every other token; doubling context quadruples cost. Fundamental.
2. **KV cache memory** — grows linearly with context, layers, heads. Often the practical wall.
3. **RoPE generalization** — model behavior degrades past trained position ranges. Solvable (YaRN, position interpolation, NTK scaling).
4. **Training data scarcity at long contexts.**

RoPE compute itself is *not* a bottleneck.

### Cache invalidation

Editing any earlier token invalidates the entire KV cache from that point forward — the causal chain is broken. This is why chat UIs have to re-prefill after edits.

### Optimizations to know by name

| Technique | Idea |
|---|---|
| **GQA** | Multiple Q heads share one K/V. Used in Llama 3. |
| **MQA** | All Q heads share one K/V. Even smaller; some quality hit. |
| **Quantized KV cache** | Store cache at 8-bit or 4-bit. Halves/quarters memory. |
| **Paged attention** | Manage cache like memory pages. (vLLM.) |
| **Sliding window** | Only cache the last N tokens. (Mistral.) |
| **Attention sinks** | Anchor tokens at start + sliding window. (StreamingLLM.) |
| **Cache eviction / compression** | Drop "unimportant" tokens from cache. |

### Practical implications for Ollama / local inference

- `num_ctx` directly controls max KV cache size — don't oversize it.
- First-prompt latency is dominated by prefill.
- Long conversations slow down progressively because each new token requires attention over a larger cache.
- OOM errors at long contexts are almost always KV cache, not weights.
- For multi-stage LLM pipelines: keeping the same context cached across stages saves prefill cost. Tearing it down and rebuilding pays full prefill every time. A real optimization lever.

---

## 11. Glossary & vocabulary

| Term | Meaning |
|---|---|
| **Token** | A sub-word unit; the atomic input to the model. |
| **Vocabulary** | The complete set of tokens the tokenizer knows. ~128K for Llama 3. |
| **Embedding** | The learned vector representation of a token. |
| **Embedding dimension** (`d_model`) | Size of token vectors. 4096 in Llama 3 8B. |
| **Logits** | Raw unnormalized scores output before softmax. |
| **Softmax** | Function that turns scores into a probability distribution summing to 1. |
| **Parameter / weight** | One single learned number in the model. |
| **Layer** | One block of attention + FFN. |
| **Hidden state** | The vector representation at a given layer / position. |
| **Q / K / V** | Query / Key / Value — three role-specific projections of a token vector used by attention. |
| **Attention head** | One parallel attention computation. Multiple heads per layer. |
| **GQA** | Grouped-Query Attention — multiple Q heads share K/V. |
| **MHA** | Multi-Head Attention — every Q head has its own K/V. Classic. |
| **MQA** | Multi-Query Attention — all Q heads share one K/V. |
| **Residual / skip connection** | Adding a sub-layer's input back to its output. Enables deep stacking. |
| **LayerNorm / RMSNorm** | Per-layer normalization. RMSNorm is the modern variant Llama uses. |
| **Causal mask** | Restriction that token at position i can only attend to positions 1..i. |
| **RoPE** | Rotary Position Embedding. Rotates Q and K vectors by position-dependent angles. |
| **SwiGLU** | The gated FFN variant Llama uses. Three matrices instead of two. |
| **LM head** | Final matrix that converts the hidden state into per-token logits. |
| **Logit lens** | Interpretability technique — apply the LM head at intermediate layers to see what the model "thinks" so far. |
| **Prefill** | Initial forward pass processing the entire prompt. Compute-bound. |
| **Decode** | Generating tokens one at a time. Memory-bandwidth-bound. |
| **KV cache** | Stored K and V vectors for past tokens, reused during decode. |
| **FLOPs (lowercase s)** | Total floating-point operations (count). |
| **FLOPS (uppercase S)** | Floating-point operations per second (rate). |
| **Temperature** | Sampling parameter that sharpens (< 1) or flattens (> 1) the output distribution. |
| **Top-k / Top-p** | Sampling strategies that restrict the candidate pool. |
| **Greedy decoding** | Always pick the highest-probability token. Deterministic. |
| **Autoregressive** | Generation strategy where each output token becomes part of the input for the next. |
| **Fine-tuning** | Continuing training on new data to adjust weights for a specific task or style. |
| **Quantization** | Reducing numerical precision (fp16 → int8 → int4) to shrink model / cache size. |

---

## 12. Cheat sheet

```
EVERY TOKEN'S JOURNEY:
  token id  →  embedding  →  + position  →  [Layer 1...32: attention then FFN]  →  final vector
  final vector  →  LM head  →  logits  →  softmax  →  sample  →  next token

KEY OPERATIONS:
  Attention scoring     = dot product of Q and K
  Attention weights     = softmax of scaled scores
  Attention output      = weighted sum of V vectors
  FFN                   = expand → activation → squeeze
  Position (RoPE)       = rotate Q, K by position-dependent angles inside attention

KEY MATRICES (all learned, frozen at inference):
  Embedding table       — what each token "means"
  W_Q, W_K, W_V         — project to query / key / value (per head, per layer)
  W_O                   — combines heads' attention output
  W_gate, W_up, W_down  — FFN with SwiGLU (per layer)
  LM head               — final projection to vocab-sized logits

WHY THINGS WORK THE WAY THEY DO:
  Why sub-word tokens?       — handle any text with finite vocab
  Why positional encoding?   — without it, sequence order is invisible
  Why RoPE specifically?     — relative position falls out naturally; generalizes to longer sequences
  Why Q/K/V (3 projections)? — let same token play different roles (asker / advertiser / deliverer)
  Why softmax?               — turn arbitrary scores into proper probabilities
  Why multi-head?            — capture multiple kinds of relationships in parallel
  Why expand in FFN?         — room to compute non-linear functions before squeezing back
  Why residual connections?  — stable gradient flow through deep stacks
  Why causal mask?           — prevents cheating during training; required for autoregressive use
  Why KV cache?              — avoids O(N²) regeneration; turns decode into O(1) per token

PRACTICAL NUMBERS (Llama 3 8B):
  Embedding dim:        4096
  Layers:               32
  Attention heads:      32 Q heads, 8 KV heads (GQA)
  FFN expansion:        4096 → 14,336 → 4096
  Vocabulary:           ~128K tokens
  Parameters:           ~8B (≈ 70% in FFN, 17% attention, 13% embed/LM head)
  KV cache per token:   ~131 KB at fp16
  Per-token FLOPs:      ~2 × params ≈ 16 billion

PHASES:
  Prefill — parallel, compute-bound. Fast per token.
  Decode  — sequential, memory-bound. ~constant time per token.

GENERATION CONTROLS (Ollama):
  temperature  — sharpness of the distribution (0 = greedy, 1 = neutral, >1 = creative)
  top_k        — keep only top-k candidates before sampling
  top_p        — nucleus sampling threshold
  num_ctx      — max context length → directly controls KV cache memory
```

---

## 13. Personal Q&A log

Captured from the live session so the original lines of inquiry are preserved.

### Q1. Is positional encoding an algorithm that looks at my input?

**No.** It's a deterministic stamp based purely on position number. Position 3 always gets the same signal regardless of what token sits there. Token "cat" always gets its own embedding regardless of position. The two are combined to produce a position-aware vector. No intelligence involved — just mechanical superposition (sinusoidal) or rotation (RoPE).

### Q2. When we rotate vectors in RoPE, is the cat vector "reset" to its original position to compare with the dog vector?

**No.** Both vectors stay rotated. The dot product math has a property that the **difference** in their rotations is what determines the similarity. Nothing is undone. The rotated vectors *are* the working Q and K vectors for that layer.

### Q3. So when we rotate by position, aren't we changing the similarity between tokens? Maybe that's the feature?

**Yes — that's exactly the feature.** Two tokens' similarity now depends on both *what* they are and *how far apart* they are. This is desirable because in language, relationships between words depend strongly on distance. RoPE bakes both into the geometry.

### Q4. In a 10K-token prompt, RoPE runs 10K times and propagates to next layers? Is RoPE a limiting factor for long context?

RoPE runs for every token, every layer (inside attention). For Llama 3 8B at 10K tokens: ~320K rotations per forward pass. But **RoPE compute is dirt cheap** — not the bottleneck. The real long-context limits are:

1. Attention's N² compute (fundamental).
2. KV cache memory (the practical wall).
3. RoPE *generalization* past trained position ranges (real but increasingly solved).

So your intuition pointed at a real problem — RoPE's behavior at unseen position ranges — but not the mechanism you thought (cost). Compute cost isn't the issue.

### Q5. What are W_Q, W_K, W_V?

Learned matrices of numbers (~16.8M each in Llama 3). Three different "lenses" that project the same input vector into three role-specific versions: a question, an advertisement, a payload. They're learned during training, frozen at inference. The same matrices apply to every input — context-dependence comes from context-dependent inputs, not from the matrices themselves adapting.

### Q6. How does the model pick which (Q, K) pairs get high attention?

Nobody picks. The dot product `q · k` scores **every pair**, and softmax converts scores to weights — the highest-scoring pairs naturally dominate. The interesting question is *why* some pairs score high: because the W_Q and W_K matrices were trained jointly such that productive matches (e.g., determiner-after-preposition needing the preposition's context) end up with Q and K vectors pointing in geometrically aligned directions. Alignment is learned, not programmed.

### Q7. Is the new V vector for a token just a scalar product of weights with each V vector, plus vector addition?

**Yes — exactly.** Each attention weight (scalar) multiplies its corresponding V vector. The resulting scaled vectors are summed element-wise. The result is a single new vector of the same dimensionality, geometrically a "weighted center of mass" pulled toward the highest-weight V vector.

### Q8. Is the prediction just the closest token in the new vector space?

Yes conceptually, with two refinements:
- It's the *final* layer's output (after all 32 layers), not the first layer's.
- It's expressed as a *probability over the whole vocabulary*, not a single nearest neighbor. The LM head dot-products the final vector against every token's signature; softmax converts to probabilities; sampling picks one.

### Q9. Is v_ computed for every token the same way?

**Yes — all tokens are updated in parallel.** Each token forms its own query, scores it against all keys (subject to causal masking), and produces its own weighted sum of values. Same mechanism, different inputs, run simultaneously.

### Q10. What does W_up/W_down mean mathematically? Are they constants? Context-dependent?

Matrices of learned numbers. Mathematically: linear transformations from one vector space to another (4096 → 14,336 and back). During training, they change. At inference, they are **frozen constants**. Their *outputs* are context-dependent because their *inputs* are context-dependent — not because the matrices themselves adapt to context.

### Q11. So a parameter or weight is every number in W_K, W_Q, W_V, W_up, W_down etc. across all 32 layers?

**Yes.** A parameter = one number inside any learned matrix. "8B parameters" = 8 billion such numbers across the embedding table, all attention matrices (Q/K/V/O), all FFN matrices (gate/up/down), all norm scale factors, and the LM head. Roughly 70% live in FFN, 17% in attention, 13% in the embedding + LM head.

### Q12. What is a FLOP?

A FLoating-point OPeration — one arithmetic operation (add, subtract, multiply, divide) on decimal numbers. **FLOPs** (lowercase s) = a count; **FLOPS** (uppercase S) = a rate (per second). Useful rule: per-token forward pass takes ~2 × number-of-parameters FLOPs. For Llama 3 8B that's ~16 billion FLOPs per token.

### Q13. (Deep dive: KV cache)

See [Section 10](#10-kv-cache--deep-dive). Short version: caches K and V for past tokens so each new generated token costs O(1) instead of O(N) recomputation. Enabled by causal masking (past K/V never change). Grows linearly with context; can exceed model weights at long context. Different from prefill (compute-bound) vs decode (memory-bandwidth-bound) phases of inference.

---

*End of reference.*
