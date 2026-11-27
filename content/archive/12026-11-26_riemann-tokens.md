+++
title = "How much did proving the Riemann Hypothesis cost?"
path = "riemann-tokens"
date = "2026-11-26"
+++

# How much did proving the Riemann Hypothesis cost?

As neither Levent Alpöge nor Anthropic have shared details on the compute costs of developing [Alpöge's Claude-written proof
of the Riemann Hypothesis](https://en.wikipedia.org/wiki/Alp%C3%B6ge's_proof_of_the_Riemann_hypothesis), I
wanted to do some basic back-of-the-envelope estimates.

The Lean proof itself is 1.4 GiB (unzipped), 175 190 files and a total of 17 373 992 lines.

If you or I wanted to recreate this proof with Claude Code, how much would it cost?

## Estimate 1: Just output tokens

Let's start off simple, and make the assumption that our Claude session managed to one-shot it, outputting *nothing* but the completed
proof. Using [Anthropic's token counting API](https://platform.claude.com/docs/en/build-with-claude/token-counting), we can see that
this would cost:

| Model      |   Output tokens |   API Cost |
|:-----------|----------------:|-----------:|
| Sonnet 5.5 |     614 002 830 |  $6 140.03 |
| Opus 5.5   |     614 002 830 | $12 280.06 |
| Fable 5.2  |     614 002 830 | $24 560.11 |
| Mythos 5.2 |     614 002 830 | $24 560.11 |
| Mythos 5   |     614 002 830 | $30 700.14 |

(Note: I'm leaving the output tokens column, even though all of those models apparently use the same tokenizer, just in case I ever
get around to adding some non-Anthropic models.)

## Estimate 2: Output + estimated reasoning

[Artificial Analysis' Token Use table](https://artificialanalysis.ai/models/claude-fable-5-2#token-use) gives us empirical ratios
between reasoning and output tokens, based on whatever mix of tasks they've selected. Unfortunately they don't have data on Mythos[^1].

If we assume max reasoning, we get:

| Model      |   Output tokens | Reasoning tokens |   API Cost |
|:-----------|----------------:|-----------------:|-----------:|
| Sonnet 5.5 |     614 002 830 |    1 602 829 176 | $22 168.32 |
| Opus 5.5   |     614 002 830 |    1 399 405 538 | $40 268.17 |
| Fable 5.2  |     614 002 830 |    1 459 376 619 | $82 935.18 |

## Estimate 3: Adjusted output + estimated reasoning

When a model writes a file, it's usually a very iterative process, writing a few sections, reviewing, compiling, editing, then writing the next sections, etc. Our estimates so far assumed that the final `.lean` files' contents were the only output tokens produced.

How much overhead does this agentic file-editing process add? I'm not going to do careful experiments here, but I'm pretty sure it's
less than 10× and more than 1.5× – let's say 2× to be safe? Since the output:reasoning ratios above were calculated
using *actual* output tokens, we need to adjust both columns:

| Model      |   Output tokens | Reasoning tokens |    API Cost |
|:-----------|----------------:|-----------------:|------------:|
| Sonnet 5.5 |   1 228 005 660 |    3 205 658 352 |  $44 336.64 |
| Opus 5.5   |   1 228 005 660 |    2 798 811 076 |  $80 536.33 |
| Fable 5.2  |   1 228 005 660 |    2 918 753 238 | $165 870.36 |

## Estimate 4: Wildly guessing

Obviously the core assumption of all the above estimates – that the model will produce a working proof of the Riemann Hypothesis on
the first try – can't possibly be right. If this proof was anything like OpenAI's Navier-Stokes proof, then Alpöge used swarms of
agents working in parallel on various directions. On the other hand, it sounds like Levent Alpöge directed and talked with the model
more personally than the OpenAI mathematicians did. Without more details on Alpöge's harness and methods, any further estimates are nothing but wild guesses.

We've also completely ignored
  - input tokens (1/5th the price of output tokens),
  - token cache writes (1/4th the price of output tokens),
  - token cache reads (1/100th or 1/200th, but can be paid many times per token),
  - the difference between the API cost and the marginal cost to Anthropic of letting Alpöge use GPUs for this,
  - and the fact that he used an internal, presumably larger model.

So let's give it our best wild guess.

Does a 50× multiplier sound too high? Too low? I can't tell, and keep coming up with arguments for both sides.

What about 100×? That would put Fable 5.2 costs near \\$17M, which would be
similar to Sam Altman's ["on the order of \\$10M" claim](https://youtu.be/iMsUcCYNKqY?t=1078) for Navier-Stokes, although that
was probably referring to internal costs, not public API costs, so maybe we double that again?

I guess my final estimate would be somewhere between \\$5M and \\$50M, for external costs.

Strange, that's what my guess was before any of this.


# Appendix: Hand-written script

Because I'm feeling somewhat fed up with AI, and the way it's changing my life, hobbies, career, mind and dreams, I decided to
write the code to do this myself the way I loved to program: Without AI assistance, in Neovim, thinking hard.

It felt nice looking things up in the official docs, Linux man pages and StackOverflow et al.

For some awful reason, I chose to do this in Bash, rather than in something that's a little easier to debug and make performant.
I'm sure there are many people out there who could do so just as quickly as I could in Python – and Claude certainly could – but I'm not one of those people.
In any case, here you go:

```bash
#!/usr/bin/env bash

shopt -s globstar nullglob dotglob

CACHE="results.txt"
MAX=100000
SML=50000

split_if_large() {
  local file="$1"

  # Skip if not a file or if smaller than a safish byte count
  if [[ $(rg -F "$file" "$CACHE") ]] || [[ ! -f "$file" ]] ||
     (( $(stat -c %s -- $file) < $SML )); then
    return
  fi

  json_escaped_contents=$(jq --raw-input --slurp '.' < $file)

  if (( $(echo $json_escaped_contents | wc -c) > $MAX )) then
    split -C $SML --suffix-length=4 -d "$file" "$file-split-"
    echo "Successfully split $file into chunks" && rm "$file"
  fi
}

get_num_tokens() {
  local file="$1"

  if [[ -f "$file" ]] && [[ ! $(rg -F "$file" $CACHE) ]] then
    escape_all_contents=$(jq --raw-input --slurp '.' < $file)

    curl https://api.anthropic.com/v1/messages/count_tokens \
      --silent \
      -H "x-api-key: $ANTHROPIC_API_KEY" \
      -H "content-type: application/json" \
      -H "anthropic-version: 2026-10-21" \
      -d '{"model":"claude-opus-5-5","system":"","messages":[
          {"role":"user","content": '"$escape_all_contents"'}
          ]}' | jq "\"$file: "'''\(.input_tokens)"' >> $CACHE
  fi
}

touch "$CACHE"

# Download and extract the LEAN proofs of the Riemann Theorem
if [[ ! -d lean/ ]]; then
  curl https://alpo.ge/lean.tar.gz.part{01..14} > lean.tar.gz
  tar -xzvf lean.tar.gz && rm lean.tar.gz
fi

# Split any files that are too big for curl when json encoded
for file in lean/**/*; do split_if_large "$file" & done; wait

echo "Done splitting files too big to send in a single curl."

# Use the Anthropic API to count all file's input token count
for file in lean/**/*; do get_num_tokens "$file" & done; wait

awk '{s+=$2} END {print "Opus 5.5 output tokens: " s}' $CACHE
```

You may have noticed the beautiful right margin in this otherwise poor Bash code.

For years now[^2], I've left the occasional block of manually justified comments in code I work on, if I was particularly pleased
with the accompanying code and had the time to do so.

If hand coding is now reserved for my free-time, and primarily serves my entertainment[^3], there's no reason to go fast, so I took
the time to refactor the script until as many of the lines as possible were precisely the same length (61 characters, for no reason).

Please do not think that this is my idea of good code, and please only judge me a little for how dumb it is. I backed myself into
corners once I was particularly pleased with certain lines' lengths – I should have used `curl`'s ability to read whole files instead
of passing them as Bash arguments, and I definitely shouldn't just fire off 175k parallel HTTP requests and hope they all come back.

I have spent too long on this already, and won't be fixing the issues. Instead, I just ran it repeatedly until I had all the token counts.

I will not be sharing how long I spent on this :) [^4]

-----

[^1]: Am I the only one who finds this strange? I'm sure Anthropic could give them access to Mythos if Anthropic *wanted* to. Is something about its performance, speed or cost embarrassing? Does it contradict their narrative on its cybersecurity abilities?

[^2]: Ever after being introduced to [this famous, manually justified masterpiece](https://gamefaqs.gamespot.com/snes/588741-super-metroid/faqs/10114).

[^3]: Although I actually think we'll be turning to it more and more for skill maintenance.

[^4]: My wife has heard about this script, which gives an uncomfortable lower bound on how long I spent. Then again, I mostly just wanted her to see and admire the right margin.
