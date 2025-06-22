# The Jedi's Guide to Mastering LogQL 🧙‍♂️

So, you've started using Grafana Loki and feel like you're trying to pilot the Millennium Falcon with the instruction manual written in Shyriiwook. You're not alone. LogQL, Loki's query language, is incredibly powerful but can feel a bit... cryptic at first.

This guide will take you from a Youngling to a Jedi Master, ready to slice and dice your logs with precision.

---
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

---

## 🛠️ Part 1: The Basics - Your First Lightsaber

Every LogQL query has three fundamental parts:

1.  **Log Stream Selector (`{...}`)**: This is how you choose *which logs* to look at. Think of it as telling R2-D2 which ship's logs to access.
2.  **Time Range (`[...]`)**: How far back in time you want to look.
3.  **Filter (`|=` or `|`)**: What you're searching for *inside* the logs.

Let's break down a simple query:

```logql
{app="deathstar"} |= "error" [5m]
```

-   `{app="deathstar"}`: This is the **Log Stream Selector**. We're only looking at logs from the application labeled `deathstar`. Labels are key-value pairs that you define when you set up your logging agent (like Promtail). **The more specific your labels, the faster your queries.**
-   `|= "error"`: This is a **Line Filter**. It searches for the word "error" anywhere in the log line.
-   `[5m]`: This is the **Time Range**. We're looking at the last 5 minutes of logs.

---

## 🚀 Part 2: The Next Step - Parsing JSON with `| json`

A lot of modern logs are structured as JSON. Plain-text searching with `|=` works, but it's like using a club instead of a lightsaber. It's clumsy.

Enter the `| json` parser. This transforms your JSON log line into a set of fields you can query directly.

**Example Log Line:**
```json
{
  "level": "error",
  "message": "Alderaan is not a valid target.",
  "request_id": "123-abc",
  "function": "fire_laser"
}
```

Instead of doing `|= "error"`, you can be much more precise:

```logql
{app="deathstar"} | json | level = "error"
```

-   `| json`: This tells LogQL, "Hey, treat the log line as JSON."
-   `| level = "error"`: This is a **Label Filter**. After parsing, `level` becomes a temporary label you can filter on. This is much more accurate because it will only match the value of the `level` field, not just the word "error" appearing anywhere.

---

## ✨ Part 3: The Jedi Power - Aggregation with `sum by(...)`

This is where you truly harness the Force. Aggregation lets you count, sum, and analyze your logs to create dashboards and alerts. This is what separates the Padawans from the Masters.

Let's look at a real-world query I wrote and break it down.

```logql
sum by(hand_type) (
  count_over_time(
    {job="main/gtoglueb2c"} 
    | json 
    | event="Strategy lookup completed" 
    | request_id!=""
    [1m]
  )
)
```

This looks intimidating, but let's dismantle it piece by piece, from the inside out.

1.  **`{job="main/gtoglueb2c"} | json ... [1m]`**: We start by selecting logs from a specific job over the last minute and parsing them as JSON.
2.  **`count_over_time(...)`**: This is a **Metric Query function**. It counts the number of log entries for each log stream within the given time range.
3.  **`| request_id != ""`**: This is a crucial filter. In this application, I noticed that some logs didn't have a `request_id`. This filter ensures we only count logs that are part of an actual request, giving us one unique log line per `request_id`. This is useful when a single `request_id` might be associated with multiple series.
4.  **`sum by(hand_type) (...)`**: Finally, we group these counts by the `hand_type` (e.g., "Flush", "Straight") to see how many times each hand type was used in total.

**In plain English, this query answers:**
> "In the last minute, how many times was each `hand_type` associated with the 'Strategy lookup completed' event, summed across all users?"

---

### ⚠️ Why Vector Matching Fails in Some Operations

Have you ever tried to divide two aggregated results and gotten an error?

```logql
# This query will fail!
sum by(stormtrooper_rank) ( 
  count_over_time({app="deathstar"} | json | event="core ready to fire" [1m])
)
/
sum (
  count_over_time({app="deathstar"} | json | event="core ready to fire" [1m])
)
* 100
```

This query is invalid in Loki because Loki doesn't support directly dividing one vector by another unless **the labels match perfectly** or you are dividing by a **scalar** (a single number).

-   The top part (`sum by(stormtrooper_rank)`) creates a vector with the label `stormtrooper_rank`.
-   The bottom part (`sum`) creates a result with no labels (it's aggregated to a single value but still treated as a vector with no labels).

Loki doesn't know how to match the ranks from the top vector with the single total from the bottom.

#### How to Fix It

-   **Vector ÷ Scalar → ✅ WORKS**
    ```logql
    sum by(hand_type)(...) / 123
    ```
-   **Vector ÷ Vector (with the same labels) → ✅ WORKS**
    ```logql
    sum by(hand_type)(count_over_time(...)) / sum by(hand_type)(count_over_time(...))
    ```
-   **Vector ÷ Vector (with different labels) → ❌ FAILS**
    ```logql
    sum by(hand_type)(...) / sum(...)
    ```

---

## 🛠️ Part 4: Common Patterns & Best Practices

### Aggregate after `| json`? Not Directly.

You *cannot* use metric query functions like `count_over_time` after a `| json` filter. These functions require a time series, but `| json` converts the logs into a filtered stream.

### ✅ Solution 1: Use Substring Filters (`|=`)

This is a faster, but less precise, way to filter. It checks if the text exists anywhere in the log line.

```logql
count_over_time( 
  {app="DeathStar"} 
  |= `"function":"wrapper"`
  |= `"level":"error"`
  [5m]
)
```

### ✅ Solution 2: Use `| json` with Grouping

This is slower but more precise, as it matches exact field values after parsing the JSON.

```logql
sum by (level) (
  count_over_time(
    {app="DeathStar"}
    | json
    | function = "wrapper"
    | level = "error"
    [5m]
  )
)
```

### 🔍 Comparing the Two Approaches

| Approach               | Pros                                       | Cons                                    |
|------------------------|--------------------------------------------|-----------------------------------------|
| **Substring (`|=`)**    | Faster, works on non-JSON or malformed logs| Can cause false positives, less precise |
| **JSON (`| json` + `by`)**| Precise, exact matching                  | Slower, requires valid JSON             |

---

### 🌟 Bonus Tip: Finding One Log Line Per Request

Sometimes, you'll see multiple log lines for the same event. To get accurate counts for a dashboard, you need to find a way to get **exactly one line per request**.

In many AWS services (Lambda, ECS), a log line containing `"function": "wrapper"` is a good candidate, as it often represents the final, aggregated log for a request. But this is not a universal rule.

**How to find the right filter:**
> *"Patience you must have, my young Padawan—in the logs, the truth runs deep."* 🧙‍♂️

There's no secret here. It's a skill anyone can develop: **Patience and attention**.

1.  Grab a sample of logs for a single `request_id`.
2.  Paste them into a text editor like VS Code to analyze the JSON structure.
3.  Look for a key-value pair that is **present in every request** but **not repeated** within the logs for that same request.

For example, I found that this filter gave me exactly one line per `request_id`, which was perfect for my analysis:

```logql
{job="main/gtoglueb2c"} 
| json
| request_id != "" 
| event = "Strategy lookup completed" 
| function = "main_lookup_strategy"
```

---

If you've made it this far, congratulations! 🎉 You are no longer a Padawan. You are ready to master LogQL like a true Jedi. 🧙‍♂️
