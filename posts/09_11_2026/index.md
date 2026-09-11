# Can a Forth Interpreter Act as Working Memory for an AI Agent?

I have been experimenting with a small project called `pforth-shmem`: a parallel Forth built on top of pForth and OpenSHMEM. The original idea was to expose a Forth environment where multiple processing elements can share data through SHMEM-style operations, but it turns out to be interesting for another reason too: it gives an AI agent a live interpreter.

That matters because Forth is unusually interactive. You do not have to write a complete program, run it, inspect the failure, and regenerate the whole thing. You can define one word, test the stack, define the next word, and gradually grow a working vocabulary inside the interpreter.

That led to a question:

Can a live Forth interpreter reduce the amount of context an AI agent needs?

In other words, can the interpreter become a kind of external working memory?

## What Is `pforth-shmem`?

`pforth-shmem` is a modification of pForth, a small ANS-like Forth written in C. This version adds OpenSHMEM support, giving Forth programs access to parallel PGAS-style communication.

At the low level, it exposes words like:

```forth
put
get
barrier-all
sync
pe
pes
shared
```

On top of that, it has friendlier Forth words in `fth/shmem.fth`, including:

```forth
shared-cell
shared-cells
remote!
remote@
all-reduce-sum
all-barrier
pe0?
```

So you can run a Forth interpreter across multiple PEs and interact with local and remote memory.

But for this experiment, the most important property was simpler: `pforth-shmem` is still Forth. It has an interpreter. It has a stack. It lets you define words dynamically.

## The Basic Idea

When an AI agent writes code in the usual way, it often has to keep a lot of state in its own context window:

- What functions it has written
- Which bugs it found
- What the last test output said
- Which helper routines exist
- What assumptions were already verified

That gets expensive. The conversation fills with code, logs, repeated commands, and revised files.

Forth suggests a different loop:

```forth
: u8 c@ 255 and ;
: le32@ ... ;
: pixel-addr ... ;
: pixel-energy ... ;
```

Each word is defined inside the interpreter. Once defined, it lives in the Forth dictionary. The AI does not need to resend it every time. It can just use it.

So the interpreter becomes a working environment, not merely a subprocess.

The hoped-for workflow is:

```text
define a tiny word
test it
inspect the stack
define the next word
test it
checkpoint the dictionary
continue
```

That is especially attractive for AI agents because Forth words are compact, composable, and easy to probe.

## The Seam-Carving Test

To test this, I used a small image-processing task: remove one vertical seam from a BMP file.

The test image was a 24-bit uncompressed BMP:

```text
samples/hopper.bmp
128 x 128 x 24
pixel data offset: 138
file size: 49290 bytes
```

The task was to:

1. Load the BMP.
2. Parse the header.
3. Compute a simple gradient energy per pixel.
4. Use dynamic programming to find a low-energy vertical seam.
5. Remove that seam.
6. Write a new BMP with width reduced from `128` to `127`.

The output file was validated externally as:

```text
127 x 128 x 24
file size: 49290
pixel offset: 138
```

The stride stayed `384` bytes because `127 * 3 = 381`, padded to a 4-byte boundary.

Here is the input image used for the experiment:

![Original 128 by 128 BMP test image](/posts/09_11_2026/hopper-original.png)

And here is the result after removing one vertical seam:

![Carved 127 by 128 image after one seam removal](/posts/09_11_2026/hopper-carved.png)

## Three Workflows Compared

I compared three ways an AI agent could solve the same task.

### 1. One Big File

This is the normal coding-agent move: generate a complete `.fth` file, run it, inspect output.

This has low interaction overhead, but poor incremental feedback. If something fails, the agent may need to reason over the whole file again.

### 2. Current Verbose Interactive Mode

This keeps one live Forth interpreter open and defines words incrementally. That is good.

But the transcript is noisy. pForth echoes input, prints `ok`, prints stack banners, and emits a lot of text around each interaction.

So although the definitions live in Forth, the model still sees a large transcript.

### 3. Proposed Compact Interactive Mode

This keeps the same persistent interpreter, but only returns compact observations to the model.

For example, instead of a full transcript, it can return:

```text
[[PFORTH-AGENT-STACK pe=0 depth=3 items=11 427 259 ]]
```

or simply:

```text
OK
```

Now the interpreter really acts like external memory. The agent sends a small new definition or probe, and receives a small result.

## Measurement Method

The experiment measured:

```text
input bytes sent to pForth
+
visible output bytes returned to the model
```

Then it estimated tokens as:

```text
estimated_tokens = ceil(bytes / 4)
```

This is not tokenizer-exact, but it is good enough for comparing workflows.

The benchmark script was:

```text
tools/measure_interpreter_tokens.py
```

It ran the same seam-carving workload in all three modes and validated that each produced a valid BMP.

## Results

| Mode | Approx Tokens | Bytes | Valid Output |
|---|---:|---:|---|
| One big file | 1,213 | 4,849 | yes |
| Current verbose interactive | 3,408 | 13,630 | yes |
| Proposed compact interactive | 1,171 | 4,682 | yes |

The result was pretty clear.

The current interactive mode is worse than writing one big file, because the transcript is too noisy.

But the compact interactive mode slightly beats the one-big-file approach even for a single complete run.

That is important because the real benefit should appear over repeated iteration. Once the interpreter has accumulated definitions, future experiments do not need to resend the whole program. The agent can just ask small questions:

```forth
64 64 pixel-energy agent-stack
```

or:

```forth
0 seam@ 64 seam@ 127 seam@ agent-stack
```

The live dictionary holds the program state. The model only needs the next move and the result.

## A Useful Surprise: Dictionary Checkpoints

pForth also has `SAVE-FORTH`, which can save the current dictionary image:

```forth
c" output/seam-carve-lab.dic" save-forth
```

I tested this with `pforth-shmem` and verified that saved dictionaries can be reloaded:

```sh
./pforth-shmem -n 2 -d output/agent-save-test.dic
```

The saved words survived reload, and basic SHMEM words still worked across multiple PEs.

That means an AI agent can potentially work like this:

```text
start interpreter
define words interactively
test them
save dictionary checkpoint
later reload dictionary
continue from there
```

That is much closer to a persistent computational notebook than a normal code-generation loop.

## The Main Takeaway

A persistent interpreter alone does not save tokens.

A persistent interpreter with verbose output may actually cost more.

But a persistent interpreter with compact observations can act as external working memory for an AI agent.

In this experiment:

```text
verbose interactivity: expensive
compact interactivity: promising
```

The compact mode worked because it moved accumulated program state out of the model context and into the Forth dictionary.

That suggests a broader pattern for AI coding tools:

```text
Do not just give the agent a shell.
Give it a live language runtime with compact, structured observations.
```

Forth is unusually good for this because it already wants to be used incrementally. The stack is inspectable. Definitions are tiny. The dictionary is live. The language naturally supports the kind of stepwise construction that AI agents need.

For `pforth-shmem`, that makes the interpreter more than a convenience. It becomes a possible memory substrate for the agent itself.
