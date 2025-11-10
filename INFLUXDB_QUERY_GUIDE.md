# InfluxDB Flux Query Customization Guide

This guide helps you understand and customize the Flux queries used in the Home Energy Usage dashboard.

## Understanding Your InfluxDB Data Schema

Before customizing queries, you need to understand how HomeAssistant stores data in InfluxDB.

### Finding Your Measurements

Run this query in InfluxDB Data Explorer to see all measurements:

```flux
import "influxdata/influxdb/schema"

schema.measurements(bucket: "HomeAssistant")
```

Common patterns:
- **Separate measurements by unit**: `W`, `kWh`, `A`, `V`
- **Single measurement**: `state` (with `_field` containing the metric name)
- **Domain-based**: `sensor`, `binary_sensor`, etc.

### Inspecting Tag Structure

Find all tags for a specific measurement:

```flux
import "influxdata/influxdb/schema"

schema.tagKeys(
  bucket: "HomeAssistant",
  predicate: (r) => r._measurement == "W"
)
```

Common tags from HomeAssistant:
- `entity_id`: Full entity ID (e.g., `sensor.power_kitchen_1mon`)
- `friendly_name`: Human-readable name (e.g., `Kitchen 1Mon`)
- `domain`: Entity domain (e.g., `sensor`)
- `device_class`: Type of sensor (e.g., `power`, `energy`)

### Sample Data Query

View actual data to understand structure:

```flux
from(bucket: "HomeAssistant")
  |> range(start: -1h)
  |> filter(fn: (r) => r.entity_id =~ /power/)
  |> limit(n: 10)
```

## Common Query Patterns

### Pattern 1: Get Latest Value (Instant Query)

**Use case**: Current power consumption, monthly totals

```flux
from(bucket: "HomeAssistant")
  |> range(start: -5m)                           // Look at last 5 minutes
  |> filter(fn: (r) => r._measurement == "W")    // Filter by measurement
  |> filter(fn: (r) => r._field == "value")      // Filter by field
  |> filter(fn: (r) => r.friendly_name == "Kitchen")  // Filter by device
  |> last()                                      // Get most recent value
```

**Customization:**
- Change `_measurement` to match your schema (`W`, `kWh`, `state`, etc.)
- Change `_field` if your data uses different field names
- Change `friendly_name` to match your sensor names

### Pattern 2: Filter with Regex

**Use case**: Exclude totals, filter multiple devices

```flux
from(bucket: "HomeAssistant")
  |> range(start: -5m)
  |> filter(fn: (r) => r._measurement == "W")
  |> filter(fn: (r) => r.friendly_name !~ /.*Total.*/)  // Exclude "Total"
  |> last()
```

**Regex patterns:**
- `=~` : Matches regex
- `!~` : Does NOT match regex
- `/.*Total.*/` : Contains "Total"
- `/^Kitchen.*/` : Starts with "Kitchen"
- `/.*1Mon$/` : Ends with "1Mon"

### Pattern 3: Top N Values

**Use case**: Top 5 power consumers

```flux
from(bucket: "HomeAssistant")
  |> range(start: -5m)
  |> filter(fn: (r) => r._measurement == "W")
  |> filter(fn: (r) => r._value > 5.0)          // Only devices using >5W
  |> last()
  |> top(n: 5, columns: ["_value"])             // Get top 5
```

**Alternatives:**
- `bottom(n: 5)` - Get lowest values
- `sort(columns: ["_value"], desc: true)` - Sort without limiting

### Pattern 4: Time-Based Aggregation

**Use case**: Hourly averages, daily totals

```flux
from(bucket: "HomeAssistant")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "W")
  |> aggregateWindow(every: 1h, fn: mean, createEmpty: false)
```

**Aggregation functions:**
- `mean` - Average value
- `sum` - Total/sum
- `max` - Maximum value
- `min` - Minimum value
- `last` - Last value in window
- `first` - First value in window
- `median` - Median value

**Time windows:**
- `every: 1m` - 1 minute
- `every: 5m` - 5 minutes
- `every: 1h` - 1 hour
- `every: 1d` - 1 day

### Pattern 5: Mathematical Transformations

**Use case**: Cost calculations, unit conversions

```flux
from(bucket: "HomeAssistant")
  |> range(start: -5m)
  |> filter(fn: (r) => r._measurement == "kWh")
  |> filter(fn: (r) => r.friendly_name == "Total Power 1Mon")
  |> last()
  |> map(fn: (r) => ({ r with _value: r._value * 0.0965 }))  // Convert to cost
```

**Operations:**
- `r._value * 2` - Multiply
- `r._value / 1000` - Divide
- `r._value + 10` - Add
- `r._value - 5` - Subtract
- `(r._value * 0.0965) * 1.05 + 14.0` - Complex formula

### Pattern 6: Rate of Change

**Use case**: Power spikes, consumption rate

```flux
from(bucket: "HomeAssistant")
  |> range(start: -7d)
  |> filter(fn: (r) => r._measurement == "kWh")
  |> derivative(unit: 1h, nonNegative: true)    // Change per hour
```

**Options:**
- `unit: 1s` - Change per second
- `unit: 1m` - Change per minute
- `unit: 1h` - Change per hour
- `nonNegative: true` - Ignore negative changes (for counters)

### Pattern 7: Difference/Increase

**Use case**: Energy consumed in a time period

```flux
from(bucket: "HomeAssistant")
  |> range(start: -7d)
  |> filter(fn: (r) => r._measurement == "kWh")
  |> difference()                               // Difference between consecutive points
  |> sum()                                      // Total increase
```

**Alternatives:**
- `difference(nonNegative: true)` - Only positive changes
- `increase()` - Cumulative increase (InfluxDB 2.1+)

### Pattern 8: Grouping and Aggregation

**Use case**: Sum by device, total by category

```flux
from(bucket: "HomeAssistant")
  |> range(start: -7d)
  |> filter(fn: (r) => r._measurement == "kWh")
  |> difference()
  |> group(columns: ["friendly_name"])          // Group by device
  |> sum()                                      // Sum each group
  |> group()                                    // Ungroup for display
  |> top(n: 5)
```

## Dashboard Query Examples

### Panel: Top 5 Devices Currently Running

```flux
from(bucket: "HomeAssistant")
  |> range(start: -5m)
  |> filter(fn: (r) => r._measurement == "W" or r._measurement == "state")
  |> filter(fn: (r) => r._field == "value")
  |> filter(fn: (r) => r.friendly_name !~ /.*Total.*/)
  |> filter(fn: (r) => r._value > 5.0)
  |> last()
  |> top(n: 5, columns: ["_value"])
```

**Customization tips:**
- Adjust threshold: `r._value > 5.0` → `r._value > 10.0` for devices over 10W
- Change measurement name to match your schema
- Add more exclusions: `!~ /.*Total.*|.*Balance.*/`

### Panel: Monthly Power Total

```flux
from(bucket: "HomeAssistant")
  |> range(start: -5m)
  |> filter(fn: (r) => r._measurement == "kWh" or r._measurement == "state")
  |> filter(fn: (r) => r._field == "value")
  |> filter(fn: (r) => r.friendly_name == "Total Power 1Mon")
  |> last()
```

**If your sensor has a different name:**
- Replace `"Total Power 1Mon"` with your actual sensor's friendly name
- Find it in HomeAssistant: Developer Tools → States
- Or query InfluxDB: `schema.tagValues(bucket: "HomeAssistant", tag: "friendly_name")`

### Panel: Monthly Bill Calculation

```flux
from(bucket: "HomeAssistant")
  |> range(start: -5m)
  |> filter(fn: (r) => r._measurement == "kWh" or r._measurement == "state")
  |> filter(fn: (r) => r._field == "value")
  |> filter(fn: (r) => r.friendly_name == "Total Power 1Mon")
  |> last()
  |> map(fn: (r) => ({ r with _value: (r._value * 0.0965) * 1.05 + 14.0 }))
```

**Customize for your utility:**

Example 1 - Simple rate:
```flux
|> map(fn: (r) => ({ r with _value: r._value * 0.12 }))  // $0.12/kWh
```

Example 2 - Rate + tax:
```flux
|> map(fn: (r) => ({ r with _value: (r._value * 0.0965) * 1.08 }))  // 8% tax
```

Example 3 - Tiered pricing:
```flux
|> map(fn: (r) => ({
    r with _value:
      if r._value <= 500.0 then r._value * 0.10
      else if r._value <= 1000.0 then (500.0 * 0.10) + ((r._value - 500.0) * 0.12)
      else (500.0 * 0.10) + (500.0 * 0.12) + ((r._value - 1000.0) * 0.15)
}))
```

### Panel: Hourly Power by Device

```flux
from(bucket: "HomeAssistant")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "W" or r._measurement == "state")
  |> filter(fn: (r) => r._field == "value")
  |> filter(fn: (r) => r.friendly_name !~ /.*Total.*/)
  |> aggregateWindow(every: 1h, fn: mean, createEmpty: false)
  |> map(fn: (r) => ({ r with _value: r._value / 4.0 }))
```

**Customization:**
- Change aggregation: `fn: mean` → `fn: max` for peak usage
- Change window: `every: 1h` → `every: 30m` for 30-minute intervals
- Remove division if not needed: Delete the `map()` line

### Panel: Top 5 Devices Last Week

```flux
from(bucket: "HomeAssistant")
  |> range(start: -7d)
  |> filter(fn: (r) => r._measurement == "kWh" or r._measurement == "state")
  |> filter(fn: (r) => r._field == "value")
  |> filter(fn: (r) => r.friendly_name !~ /.*Total.*|.*Mon.*/)
  |> difference()
  |> group(columns: ["friendly_name"])
  |> sum()
  |> group()
  |> top(n: 5, columns: ["_value"])
```

**Customization:**
- Change time range: `start: -7d` → `start: -30d` for monthly view
- Change top N: `n: 5` → `n: 10` for top 10

## Troubleshooting Common Issues

### Issue: No data showing

**Check 1 - Verify measurement name:**
```flux
import "influxdata/influxdb/schema"
schema.measurements(bucket: "HomeAssistant")
```

**Check 2 - Verify field name:**
```flux
import "influxdata/influxdb/schema"
schema.fieldKeys(
  bucket: "HomeAssistant",
  predicate: (r) => r._measurement == "W"
)
```

**Check 3 - View raw data:**
```flux
from(bucket: "HomeAssistant")
  |> range(start: -1h)
  |> limit(n: 10)
```

### Issue: Wrong values displayed

- Check units: Is your sensor in W or kW? Wh or kWh?
- Check calculations: Verify the math in `map()` functions
- Check time range: Is `start: -5m` appropriate for your data frequency?

### Issue: Sensor name doesn't match

Find exact sensor names:
```flux
import "influxdata/influxdb/schema"

schema.tagValues(
  bucket: "HomeAssistant",
  tag: "friendly_name",
  predicate: (r) => r._measurement == "W"
)
```

Update queries with exact names from the result.

## Advanced Techniques

### Combining Multiple Measurements

If you have both `W` and `kWh` measurements:

```flux
power = from(bucket: "HomeAssistant")
  |> range(start: -5m)
  |> filter(fn: (r) => r._measurement == "W")
  |> last()

energy = from(bucket: "HomeAssistant")
  |> range(start: -5m)
  |> filter(fn: (r) => r._measurement == "kWh")
  |> last()

union(tables: [power, energy])
  |> yield(name: "combined")
```

### Using Variables

Grafana dashboard variables in Flux:

```flux
from(bucket: "HomeAssistant")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r.friendly_name == "${device_name}")
```

### Conditional Filtering

```flux
from(bucket: "HomeAssistant")
  |> range(start: -1h)
  |> filter(fn: (r) =>
      r._measurement == "W" and
      (r.friendly_name == "Kitchen" or r.friendly_name == "Living Room")
  )
```

## Resources

- [Flux Documentation](https://docs.influxdata.com/flux/v0.x/)
- [Flux Standard Library](https://docs.influxdata.com/flux/v0.x/stdlib/)
- [InfluxDB Query Examples](https://docs.influxdata.com/influxdb/v2/query-data/flux/)
- [Grafana Flux Support](https://grafana.com/docs/grafana/latest/datasources/influxdb/query-editor/)

## Quick Reference: PromQL to Flux Translation

| PromQL | Flux Equivalent |
|--------|-----------------|
| `metric{label="value"}` | `filter(fn: (r) => r.label == "value")` |
| `metric{label=~"regex"}` | `filter(fn: (r) => r.label =~ /regex/)` |
| `metric{label!~"regex"}` | `filter(fn: (r) => r.label !~ /regex/)` |
| `topk(5, metric)` | `top(n: 5, columns: ["_value"])` |
| `avg_over_time(metric[1h])` | `aggregateWindow(every: 1h, fn: mean)` |
| `rate(metric[1h])` | `derivative(unit: 1h, nonNegative: true)` |
| `increase(metric[1h])` | `difference() \|> sum()` |
| `sum by (label)` | `group(columns: ["label"]) \|> sum()` |
| `metric > 5` | `filter(fn: (r) => r._value > 5.0)` |
| `metric * 2 + 10` | `map(fn: (r) => ({r with _value: r._value * 2.0 + 10.0}))` |

---

Happy querying! If you need help with specific customizations, check the InfluxDB or Grafana documentation linked above.
