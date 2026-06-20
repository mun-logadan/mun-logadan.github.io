+++
title = "pass@k vs. pass^k"
path = "pass-to-the-k"
date = "2026-06-21"
+++

# pass@k vs. pass^k

I recently saw a blog post defining formulas for `pass@k` and `pass^k` and get it wrong.

The natural language summary of each is:
  - `pass@k`: The chance an experiment run $k$ times will succeed at least once
  - `pass^k`: The chance an experiment run $k$ times will succeed every time

So `pass@k` is good for quantifying low success rates, and `pass^k` is good for
quantifying high success rates[^1].

You'll mostly see these reported in various LLM benchmarks –
[τ²-bench](https://github.com/sierra-research/tau2-bench) was where I first saw
`pass^k`.

The wrong and right ways to calculate each of these are just two different sets
of estimators for them, each with their own tradeoffs.

## Wrong formulas (biased estimators)

These are honestly what I might have implemented, had they been my ideas.
They're quite intuitive looking, with

$$
\begin{align}
    \text{pass@k} &= 1 - \left(\frac{n - s}{n}\right)^k, \\\\[8pt]
    \text{pass^k} &= \left(\frac{s}{n}\right)^k,
\end{align}
$$
where $n$ is the number of trials you ran in real life and $s$ was the number of successes.

In other words: After having run $n$ trials, you take $\frac{s}{n}$ to be the
true rate of success, and set `pass@k` to the chance that $k$ trials won't all
fail and `pass^k` to the chance that all will succeed.

## Right formulas (unbiased estimators)

The above formulas sound pretty convincing, but unfortunately they're biased
estimators, undershooting `pass@k` and overshooting `pass^k` for small $n$. The
more trials you run, the more accurate they become, but as is often the case,
your research budget might not allow you to get to that point.

Instead, you can (and any research paper should![^2]) use the unbiased estimators
$$
\begin{align}
    \text{pass@k} &= 1 - \frac{n - s \choose k}{n \choose k}, \\\\[8pt]
    \text{pass^k} &= \frac{s \choose k}{n \choose k}.
\end{align}
$$

These can be read as:
  - The ratio between the number of ways to choose $k$ failures out of your
    original $n$ trials and the number of ways to choose any $k$ trials out of
    them
  - The ratio between the number of ways to choose $k$ successes out of your
    original $n$ trials and the number of ways to choose any $k$ trials out of
    them

Because they're unbiased, any error in your estimates of `pass@k` and `pass^k`
will be evenly distributed above and below the true values. In exchange, you
will have more variance, but as these are usually reported in the aggregate
over dozens to hundreds of tasks, this variance will drop quickly, unlike bias.

## Further reading

For a nice explanation of why the `pass@k` formula is biased, with nice charts and some more applications, see [Han Lee's blog post](https://leehanchung.github.io/blogs/2025/09/08/pass-at-k/).

For a numerically stable implementation of `pass@k` (binomial coefficients get very large, if you have high enough $n$), you can read [the paper that introduced `pass@k`](https://arxiv.org/abs/2107.03374).

For the paper that introduced `pass^k`, [look here](https://arxiv.org/abs/2406.12045).

----

[^1]: By "good for quantifying low/high success rates", I mean that they help spread out probabilities tightly clustered around $0.0$ or $1.0$.  
It's more about human psychology and helping us relate probabilities to the real world than anything else.

[^2]: Barring some very specific circumstance, and only if accompanied by very strong reasons for and explanations of that circumstance, I'm sure.
