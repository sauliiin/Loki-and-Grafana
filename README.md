# 🚀 LogQL: From Padawan to Jedi in the World of Logs - An ongoing project builded in order like star Wars Movies 😜

If you're using Loki to explore logs more precisely — especially with AWS — this guide will take you from the basics to some handy advanced tricks. 😎

## 🧪 First things first: Loki is *pipe-based* (`|`)

> But what are *pipes*?

They're just vertical bars `|` that let you chain filters together. Think of them as a way to pass results from one stage into the next.

### Example:

```logql
request_id != "" | hand_type = "cash-game"
```

This filters logs where `request_id` is not empty, then applies a second filter for `hand_type = "cash-game"`. Each stage narrows the results.

## ✍️ Writing clean filters

You can write like this (the app="deathstar" is the name of the app I want to obtain data, its is selected over 
`Label browser` when you are building you dashboard):

Obs: If your logs are seen at loki as json files, simple put `| json` before using the filters. If you tipe slowly, you will see hinst for witch terms you can use.

```logql
{app="deathstar"} | json | function = "wrapper" | level = "error"
```

Or like this (same effect, easier to read - in my opinion):

```logql
{app="gtoglueb2c"}
| json
| function = "wrapper" 
| level = "error"
```

Even with all that, you may still see multiple log lines for example, that look exatcly the same. This happens when the system logs both the beginning and end of a process, or from different modules. You can sometimes de-duplicate by logic (if it is exctle 2 duplicated lines, simply divide the result by 2) — or (recommended) use better filters 👇

## 💡 TIP: Look out for `"function": "wrapper"`

In AWS logs (like Lambda, API Gateway, ECS) is is common to find a line that end with `"function": "wrapper"`, this usually means:

- It’s a high-level middleware or handler
- It’s the **final** step of the request lifecycle
- It aggregates all the info: status, exceptions, timing, etc.

So filtering for `"function": "wrapper"` can give you **one log line per request**, which is great for dashboards. But, you have to double check that! Tometimes, even with that filter one, deppending on your log, you may have more than onle line. If so, look for another key work

Ok, But where do I look fot the `"function": "wrapper"`, or how do I understand how loki handle my data? 

There is no screet here. Its basically a soft skill that anyone can develop: Patience and antention... You need to ckech yout logs and understand them. Pick a good sample, use VS Code tho have a clean look at them, and try to identify if some line, for example are repeated, wicht words can be used as key-words for specific filtering.

# Understanding `count`, `count_over_time`, `sum`, `sum by`, `count by` and Series in Loki (Grafana)

This document explains in simple terms how `count`, `count_over_time`, and nested queries behave in Loki's LogQL. It includes clear analogies and examples to help beginners understand the concepts.
---

## 👶 Imagine this
You have several **buckets**, one for each type of playing card (e.g., `King`, `Queen`, `Jack`, etc.).  
Every time someone plays a card, you **write it on a piece of paper and drop it into the corresponding bucket**.

---

### 📌 What does `count_over_time(...)` do?
It looks into each bucket and **counts how many pieces of paper are inside**, meaning **how many times that card was played** in the chosen time window (e.g., last 10 minutes).

**Example result:**
- `King`: 4  
- `Queen`: 2  
- `Jack`: 0

---

### 📌 What does `count(count_over_time(...))` do?
This part **doesn’t care how many papers are in each bucket**, it just wants to know:
> **How many buckets have at least 1 paper inside?**

So:
- `King`: 4 → ✅ counted  
- `Queen`: 2 → ✅ counted  
- `Jack`: 0 → ❌ ignored

**Final result = 2 buckets with paper**, so `count(...)` returns **2**.

---

### 📌 What does `count(...)` alone do?
When used on its own, `count()` just **counts how many different time series exist** — meaning **how many log groups**, usually defined by labels like `hand_type`, `job`, etc.  
If you run something like `count({app="x"})`, it tells you **how many "lines" or series there are**, but **not how many events are in each**.

---

### ✅ Connecting to your query
- `count_over_time(...)` counts **how many times** the log appeared per label (`hand_type`).
- The outer `count(...)` just counts **how many different labels had at least one log**.

---

## 🧠 Summary
| Query part                   | What it does (simple explanation)                      |
|-----------------------------|---------------------------------------------------------|
| `count_over_time(...)`      | Counts **how many times** the event happened per type.  |
| `count(count_over_time(...))`| Counts **how many types** had at least one event.       |
| `count(...)` alone          | Counts **how many log series** exist.                   |

# 🧒 Lets complicate a little bit?

This may look complicated, but don’t worry! We’ll explain it like we’re talking to a kid. 🎒

Here’s the full query:

```logql
sum by(hand_type) (
  sum (
    count_over_time(
      {app="gtoglueb2c"}
      | json
      | event = "Strategy lookup completed"
      [1m]
    )
  ) by (request_id)
)
```

---

## 🎮 What’s Happening?

Imagine you and your friends are playing a card game, and a magical notebook writes down every move. That notebook is your **log**!

Each time someone plays a move, the notebook writes:

- Who played (`request_id`)
- What type of hand they used (`hand_type`)
- What happened in the move (like `"Strategy lookup completed"`)

---

## 🤩 What the Query Does

Let’s break it down into easy steps:

### 1. 📖 Look at Recent Pages Only

```logql
[1m]
```

This means:\
“Only check the last 1 minute of logs — we don’t care about the old stuff!”

---

### 2. 🔍 Find Special Moves

```logql
| event = "Strategy lookup completed"
```

We only want to count the special moves called **"Strategy lookup completed"**.

---

### 3. 👦 Count Per Player (Request ID)

```logql
count_over_time(...)
```

This counts how many times each kid (identified by `request_id`) made that special move in the last minute.

---

### 4. ➕ Add Up Per Player

```logql
sum (...) by (request_id)
```

Now we add up how many special moves each player made.

Example:

- Ana (`request_id=1`): 2 moves
- João (`request_id=2`): 3 moves

---

### 5. 🎴 Group by Hand Type

```logql
sum by(hand_type) (...)
```

Finally, we check **what kind of hand** (like "Flush", "Straight") each player used, and count how many times **each hand type** was used.

Result might be:

- Flush: 5 times
- Straight: 2 times

---

## ✅ What This Query Tells Us

> “How many times each hand type was used in special moves, grouped by all the players, in the last 1 minute.”

It looks at all the request IDs, counts their moves, and groups the total by the kind of hand (`hand_type`).

---

## ⚠️ Aggregation after `| json`? Not directly.

You *can’t* use `count_over_time`, `sum_over_time`, etc., after using `| json` and field-based filters.

Why? Because that turns logs into a *log stream*, not a *time series* that aggregations require.

### ✅ Solution: use substring filters (`|=`)

```logql
count_over_time( 
  {app="gtoglueb2c"} 
  |= "function":"wrapper"" 
  |= "level":"error" 
  [5m]
)
```

This will:

- Filter logs from the `gtoglueb2c` app
- Match lines that contain `"function":"wrapper"` *and* `"level":"error"`
- Count how many of these appear every 5 minutes

## 🛠️ Cleaner workaround: use JSON and `sum by(...)`

```logql
sum by (level) (
  count_over_time(
    {app="gtoglueb2c"}
    | json
    | function = "wrapper"
    | level = "error"
    [5m]
  )
)
```

This:

1. Parses the log line as JSON
2. Filters only logs where `function == "wrapper"` and `level == "error"`
3. Counts them in 5-minute intervals
4. Groups and sums by the `level` value (in this case, just `"error"`)

## 🔍 Comparing the two approaches

- **Substring-based (`|=`):**
  - Matches if string appears anywhere
  - Can include false positives
  - Faster, works with malformed or multiline logs
  - Returns one time series

- **JSON-based (`| json` + field filters):**
  - Matches only if extracted fields match exactly
  - Accurate, depends on valid JSON
  - Slower but more precise
  - Returns one per `level`, `function`, etc.

## 🌟 Bonus: Filtering to one line per request_id for hand_type dashboard.

```logql
{job="main/gtoglueb2c"} 
| json
| request_id != "" 
| event = "Strategy lookup completed" 
| function = "main_lookup_strategy"
```
This gave me exactly **one line per request_id** — perfect for accurate analysis. ✅

But how did I figure that out?

Because I'm a Jedi, obviously. 😏

But I also copied some log lines into a JSON file and analyzed them in VS Code to figure out which field was **not repeated across all lines**, but also **not missing in any request_id** — that’s how I narrowed it down.

> *“Patience you must have, young Padawan — in the logs, truth hides deep.”* 🧙‍♂️
---

If you’ve made it this far — congrats! 🎉  
You’re no longer a Padawan. You’re ready to master LogQL like a true Jedi. 🧙‍♂️
