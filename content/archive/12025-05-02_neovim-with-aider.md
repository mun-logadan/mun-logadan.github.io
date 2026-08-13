+++
title = "Neovim with Aider"
path = "neovim-with-aider"
date = "2025-05-02"
+++

I've been using [aider](https://aider.chat/) at work lately (mostly for Python, once for some HTML/CSS/JS), and I appreciate it when it works well.

I'm not convinced it actually speeds me up, as opposed to my use of [Simon Willison's CLI interface for LLMs](https://llm.datasette.io/en/stable/index.html), `llm`, for asking questions, transforming snippets, and writing one-off scripts. But I assume it and similar technologies will improve, and I'd like to know how and when they do speed me up.

----

One thing aider could benefit from is better IDE integration. I say this as a hypocrite, having tried [none](https://github.com/joshuavial/aider.nvim) of the [three](https://github.com/nekowasabi/aider.vim) [plugins](https://github.com/ddzero2c/aider.nvim) that already exist for Neovim (my current editor of choice).

That being said, today I made a neat integration of aider's scripting capabilities with [Neovim's quickfix-do commands](https://neovim.io/doc/user/quickfix.html#%3Acfdo), like `:cfdo`. Assuming your quickfix list is populated with references to lines of code you'd like to fix or refactor[^populate-quickfix]:
```
:cfdo !aider-change % "Refactor all uses of SentenceBoondoggle, so that it takes a numpy array of ints instead of the current strings, tokenized with the huggingface tokenizer 'sandwich-man', which you should instantiate somewhere reasonable if needed"
```

Requisite is then this script, saved in `~/.local/bin/aider-change` or wherever else in your path you'd like
```bash
#!/bin/bash

filename=$1
prompt="$2
Only edit this file, don't ask to add any others to the chat."
model="gemini/gemini-2.5-pro-preview-03-25"  # or o4-mini, claude-3.7-sonnet, etc.

aider \
    --model $model \
    --message $prompt \
    --dirty-commit \
    --no-auto-commits \
    --no-analytics \
    $filename
```

This allows you to automate tedious, yet not easily regex/vimable refactors and changes across large numbers of files[^small-num-files]. Afterwards, I open `git difftool` to review the changes, then allow aider to commit them with `/commit` in a separate, long-running interactive aider session.

How well this performance is heavily dependent on choosing the right tasks and giving the model enough context to make consistent edits in N independent sessions (one per file).

For instance, if you know you want it to use a certain syntax, provide an example of it. You can also run it first on a couple files, then adjust the prompt until it works consistently.

----

One thing that's missing for me with this is the ability to run the aider edits in parallel. In an earlier iteration of this tooling, I used a bash script with a for loop running `aider` as background jobs. 

Still, I usually have enough to do that taking 30 seconds to go look at messages is a welcome break, as long as it only happens occasionally.


[^populate-quickfix]: E.g., with `:grep "some_pattern"` or using your LSP's `referenceProvider` support (which finds all uses of a symbol).

[^small-num-files]: If changing less than 4 to 5 files, you can just use aider in the normal interactive chat, by `/add`-ing all the files to the chat, and asking for the change. It's with 10 to 20 files that this trick becomes very useful.
