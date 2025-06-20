# 🚀 LogQL: From Padawan to Jedi in the World of Logs

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

```logql
{app="gtoglueb2c"} | json | function = "wrapper" | level = "error"
```

Or like this (same effect, easier to read):

```logql
{app="gtoglueb2c"} 
| json 
| function = "wrapper" 
| level = "error"
```

Even with all that, you may still see multiple log lines for the same `request_id`. This happens when the system logs both the beginning and end of a process, or from different modules. You can sometimes de-duplicate by logic — or use better filters 👇

## 💡 TIP: Look out for `"function": "wrapper"`

In AWS logs (like Lambda, API Gateway, ECS), this usually means:

- It’s a high-level middleware or handler
- It’s the **final** step of the request lifecycle
- It aggregates all the info: status, exceptions, timing, etc.

So filtering for `"function": "wrapper"` can give you **one log line per request**, which is great for dashboards.

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

| Substring-based (`|=`)                                                | JSON-based (`| json` + field filters)                                 |
|----------------------------------------------------------------------|------------------------------------------------------------------------|
| Matches if string appears anywhere                                   | Matches only if extracted fields equal the values                     |
| Can include false positives                                          | Accurate, depends on valid JSON                                       |
| Faster, works with bad or multiline logs                             | Slower but more precise                                               |
| Returns one time series                                              | Returns one per `level`, `function`, or other fields                  |

## 🌟 Bonus: Filtering to one line per request_id

```logql
{job="main/gtoglueb2c"} 
| json
| request_id != "" 
| event = "Strategy lookup completed" 
| function = "main_lookup_strategy"
```

This gave me exactly **one line per request_id** — perfect for accurate analysis. ✅

---

If you’ve made it this far — congrats! 🎉  
You’re no longer a Padawan. You’re ready to master LogQL like a true Jedi. 🧙‍♂️
