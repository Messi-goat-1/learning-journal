# Build log · Day 3 🔬

**Date:** October 17, 2026
**Project:** Kerneye (built on [Coroot](https://github.com/coroot/coroot))

---

## What I did

### 1. Tested Kerneye on a real app

I installed the [Docker Example Voting App](https://github.com/dockersamples/example-voting-app) (5 small services).
Kerneye found them all and drew the service map on its own, with no setup.

### 2. Fixed the empty Traces page

The page was empty, but the database had hundreds of thousands of traces.
The cause was one switch in the code: `false` for users, `true` for the AI side.

**My fix:** the page now shows traces from the Kerneye agent, with a filter for
**application** (customer apps) or **monitoring** (Kerneye itself).

Customers see traces right away, without changing their code.

---

## What I learned 📚

| Idea | In one line |
|---|---|
| **eBPF** | Small programs inside the Linux kernel that watch everything, with no code changes |
| **Span vs Trace** | A span is one step. A trace is the whole request. |
| **Profiling vs Tracing** | Profiling: which function eats the CPU. Tracing: which call is slow. |
| **Safe changes** | New branch, backend and frontend in separate commits, then a Pull Request |

---

## Mistakes and surprises 🐞

- **A "memory leak" warning** was just cache filling up. It went away on its own.
- **Redis showed 5 restarts**, but they were mine, from testing.
- **Kerneye traced itself 638,970 times.** Now it's hidden by default.

The lesson: **a warning is not always a problem.** Check the numbers first.

---

## Next step 🛡️

Back to the main goal: **the security layer**.

---

## Question for you

Have you ever chased a "bug" that turned out to be a setting someone chose on purpose?
Share your story: [@kerneyehq](https://instagram.com/kerneyehq)