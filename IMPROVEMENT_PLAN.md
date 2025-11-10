# InfluxDB Energy Dashboard - Improvement Plan

## Executive Summary

This document outlines future improvements and additional visualizations for the home energy monitoring dashboard. The current implementation provides solid real-time monitoring, cost tracking, and basic analytics. The following plan categorizes enhancements by priority and complexity.

---

## Current Dashboard Assessment

### Strengths
- ✅ Real-time power monitoring with 1-minute refresh
- ✅ Cost tracking with configurable electricity rates
- ✅ Multiple time aggregations (hourly, daily, weekly, monthly)
- ✅ Anomaly detection via power spike analysis
- ✅ Top consumer identification
- ✅ Well-documented with comprehensive guides
- ✅ Clean migration from Prometheus to InfluxDB

### Current Gaps
- ⚠️ No dashboard variables for dynamic filtering
- ⚠️ Fixed electricity rates (no time-of-use support)
- ⚠️ Limited historical analysis (7-day maximum)
- ⚠️ No alerting configured
- ⚠️ No power factor or efficiency metrics
- ⚠️ No comparative analysis (month-over-month, year-over-year)
- ⚠️ No predictive analytics

---

## Priority 1: Quick Wins (Easy Implementation, High Value)

### 1.1 Dashboard Variables
**Impact:** High | **Effort:** Low | **Timeline:** 1-2 hours

Add Grafana variables for dynamic filtering:

```yaml
Variables:
  - Name: device_filter
    Type: Query
    Query: SHOW TAG VALUES WITH KEY = "entity_id"
    Multi-select: true
    Include All: true

  - Name: time_range
    Type: Interval
    Options: [1h, 6h, 12h, 24h, 7d, 30d, 90d]
    Default: 24h

  - Name: cost_threshold
    Type: Custom
    Options: [0, 5, 10, 25, 50]
    Default: 0
    Description: Minimum daily cost to display
```

**Benefits:**
- Users can focus on specific devices
- Flexible time range selection without editing queries
- Filter out low-consumption devices dynamically

---

### 1.2 Time-of-Use (TOU) Rate Support
**Impact:** High | **Effort:** Medium | **Timeline:** 3-4 hours

Support tiered electricity pricing based on time of day:

**New Panel: "TOU Rate Indicator"**
- Type: Stat panel with current rate category
- Shows: Peak / Mid-Peak / Off-Peak / Super Off-Peak
- Color-coded for quick identification

**Rate Structure Example:**
```flux
peak_hours = (hour >= 16 and hour <= 21)       // $0.18/kWh
mid_peak = (hour >= 11 and hour <= 16)         // $0.13/kWh
off_peak = (hour >= 7 and hour <= 11)          // $0.087/kWh
super_off_peak = (hour >= 0 and hour <= 7)     // $0.065/kWh
```

**Updated Cost Calculations:**
- Modify "Current Month Bill" to use time-weighted rates
- Add "Peak vs Off-Peak Usage %" panel
- Show potential savings with optimized usage

---

### 1.3 Baseline Consumption Panel
**Impact:** Medium | **Effort:** Low | **Timeline:** 1 hour

**New Panel: "Always-On Load (Vampire Power)"**
- Type: Stat panel with sparkline
- Query: Minimum power over 24-hour period
- Shows: Baseline consumption that never drops to zero
- Calculation: `min(power) over 24h`

**Value:** Identifies phantom loads and standby power waste (typically 5-10% of total consumption)

---

### 1.4 Efficiency Score
**Impact:** Medium | **Effort:** Medium | **Timeline:** 2-3 hours

**New Panel: "Energy Efficiency Score"**
- Type: Gauge (0-100 scale)
- Factors:
  - Peak vs off-peak usage ratio (30%)
  - Baseline load percentage (20%)
  - Day-over-day consistency (20%)
  - Month-over-month improvement (30%)

**Formula:**
```
score = (
  (off_peak_kwh / total_kwh * 100) * 0.30 +
  ((1 - baseline_w / avg_w) * 100) * 0.20 +
  (consistency_factor * 100) * 0.20 +
  (improvement_factor * 100) * 0.30
)
```

---

## Priority 2: Advanced Visualizations

### 2.1 Comparative Analysis Panels
**Impact:** High | **Effort:** Medium | **Timeline:** 4-6 hours

**Panel Set: "Usage Comparison Dashboard"**

1. **Month-over-Month Comparison (Bar Chart)**
   - Show current month vs previous 12 months
   - Color-code increases (red) vs decreases (green)
   - Percentage difference annotations

2. **Day-of-Week Heatmap**
   - Type: Heatmap panel
   - X-axis: Hour of day (0-23)
   - Y-axis: Day of week (Mon-Sun)
   - Color: Average power consumption
   - Identifies usage patterns

3. **Year-over-Year Overlay (Line Chart)**
   - Overlay current year on previous year
   - Same date range (e.g., Jan-Oct 2024 vs Jan-Oct 2023)
   - Shows seasonal trends

---

### 2.2 Forecasting and Predictions
**Impact:** High | **Effort:** High | **Timeline:** 8-12 hours

**Panel: "End-of-Month Projection"**
- Uses Holt-Winters forecasting or simple linear regression
- Predicts total month consumption and cost
- Confidence intervals displayed
- Compare against historical averages

**Implementation (Flux):**
```flux
from(bucket: "HomeAssistant")
  |> range(start: -30d)
  |> filter(fn: (r) => r["entity_id"] == "vue_energy_today")
  |> holtWinters(n: 7, seasonality: 7)
```

**Panel: "Budget Tracker"**
- Set monthly energy budget (kWh or $)
- Progress bar showing % of budget used
- Projection shows if over/under budget
- Alert when approaching threshold

---

### 2.3 Load Duration Curve
**Impact:** Medium | **Effort:** Medium | **Timeline:** 3-4 hours

**Panel: "Load Duration Curve"**
- Type: Line chart
- X-axis: Time (percentage, 0-100%)
- Y-axis: Power (W)
- Sorts power values descending to show:
  - Peak demand capacity needed
  - Baseload vs variable load
  - Opportunities for demand management

**Query Pattern:**
```flux
from(bucket: "HomeAssistant")
  |> range(start: -30d)
  |> filter(fn: (r) => r._measurement == "W")
  |> sort(columns: ["_value"], desc: true)
  |> cumulativeDuration(unit: 1h)
```

---

### 2.4 Circuit/Breaker Group Analysis
**Impact:** Medium | **Effort:** Low-Medium | **Timeline:** 2-3 hours

**Panel: "Power by Circuit Group"**
- Groups devices by electrical circuit
- Useful for load balancing across phases
- Identifies overloaded circuits

**Requires:** Tagging devices with circuit IDs

---

### 2.5 Energy Sankey Diagram
**Impact:** High (Visual Impact) | **Effort:** High | **Timeline:** 6-8 hours

**Panel: "Energy Flow Diagram"**
- Visualizes energy flow from grid → total → individual devices
- Width of flows represents energy magnitude
- Interactive hover details

**Note:** May require custom Grafana plugin or external tool

---

## Priority 3: Cost Optimization Features

### 3.1 Demand Charges Tracking
**Impact:** Medium-High | **Effort:** Medium | **Timeline:** 3-4 hours

**For users with demand-based billing:**

**Panel: "Peak Demand This Month"**
- Tracks highest 15-minute average power
- Shows: Current peak, historical peaks, utility threshold
- Color-coded warnings when approaching expensive tiers

**Formula:**
```
Demand Charge = MAX(15-min avg power) × $/kW
```

---

### 3.2 Cost Breakdown by Device
**Impact:** High | **Effort:** Medium | **Timeline:** 3-4 hours

**Panel: "Monthly Cost by Device" (Table)**
- Columns: Device | kWh | Cost | % of Total | $/day avg
- Sortable by any column
- Highlights top 3 most expensive devices
- Shows cumulative percentage

**Panel: "Device Operating Cost" (Stat Grid)**
- Shows hourly/daily/monthly operating cost per device
- Helps make decisions about device replacement or usage reduction

---

### 3.3 Savings Opportunities Dashboard
**Impact:** High | **Effort:** Medium-High | **Timeline:** 6-8 hours

**Intelligent recommendations based on data:**

1. **"Shift to Off-Peak" Suggestions**
   - Identifies devices running during peak hours
   - Calculates potential savings if shifted
   - Example: "Running washer at 2 AM instead of 6 PM saves $0.45/load"

2. **"Vampire Power Waste"**
   - Lists devices with high standby consumption
   - Annual cost of leaving device plugged in
   - ROI on smart power strips

3. **"Device Replacement ROI"**
   - Compares current device cost vs efficient alternative
   - Calculates payback period
   - Example: "New fridge saves $15/month, 2-year payback"

---

### 3.4 Solar Integration (if applicable)
**Impact:** High | **Effort:** Medium | **Timeline:** 4-6 hours

**For homes with solar panels:**

**Panel Set: "Solar Production vs Consumption"**
1. Production vs consumption line chart (overlay)
2. Net energy (export to grid vs import)
3. Self-consumption ratio (%)
4. Grid independence score

**Panel: "Solar Savings Calculator"**
- Energy produced × retail rate = savings
- Net metering credits
- ROI tracking on solar investment

---

## Priority 4: Analytics and Insights

### 4.1 Statistical Summary Dashboard
**Impact:** Medium | **Effort:** Low-Medium | **Timeline:** 2-3 hours

**Panel: "Monthly Statistics" (Stat Grid)**
- Mean daily consumption
- Median (typical day)
- Standard deviation (consistency)
- Min/max days
- 95th percentile (near-peak typical usage)

**Value:** Understand consumption distribution beyond just averages

---

### 4.2 Weather Correlation
**Impact:** Medium-High | **Effort:** High | **Timeline:** 8-12 hours

**Requires:** Integration with weather data source (OpenWeatherMap, NOAA, etc.)

**Panel: "Temperature vs Energy Usage"**
- Type: Scatter plot or dual-axis line chart
- X-axis: Time
- Y-axis 1: Energy consumption
- Y-axis 2: Outdoor temperature
- Identifies HVAC impact on usage

**Advanced:** Calculate degree-days (HDD/CDD) and correlation coefficient

---

### 4.3 Anomaly Detection Dashboard
**Impact:** Medium | **Effort:** High | **Timeline:** 6-8 hours

**Enhanced spike detection:**

**Panel: "Usage Anomalies"**
- Uses statistical methods (z-score, IQR)
- Flags consumption >2 standard deviations from mean
- Table showing: Time | Device | Normal Range | Actual | Deviation %

**Panel: "Unusual Device Behavior"**
- Device running at unusual times
- Device consuming more/less than typical
- Device that should be off but is on

**Implementation:**
```flux
// Z-score anomaly detection
mean_value = mean(power)
stddev_value = stddev(power)
z_score = (power - mean_value) / stddev_value
anomaly = z_score > 2 or z_score < -2
```

---

### 4.4 Load Disaggregation (NILM)
**Impact:** High (Advanced) | **Effort:** Very High | **Timeline:** 40+ hours

**Non-Intrusive Load Monitoring:**
- Machine learning to identify individual appliances from total load
- Requires training data and algorithms
- Reduces need for per-device sensors

**Note:** This is a research-level feature requiring significant development

---

## Priority 5: Alerts and Notifications

### 5.1 Basic Threshold Alerts
**Impact:** High | **Effort:** Low | **Timeline:** 1-2 hours

**Configure Grafana Alerts:**

1. **High Power Alert**
   - Trigger: Total power > 5000W for 5 minutes
   - Action: Send notification
   - Prevents circuit overload

2. **High Cost Alert**
   - Trigger: Daily cost > $10
   - Action: Send notification
   - Helps stay within budget

3. **Always-On Alert**
   - Trigger: Device power > 50W at 3 AM
   - Action: Send notification for investigation

4. **Device Stuck On**
   - Trigger: Device power > threshold for 24+ hours
   - Example: Space heater left on

---

### 5.2 Predictive Alerts
**Impact:** Medium | **Effort:** Medium-High | **Timeline:** 4-6 hours

**Smart notifications:**

1. **Budget Overrun Warning**
   - "You're on track to exceed your $150 budget by 15%"
   - Sent at 50%, 75%, 90%, 100% of month

2. **Peak Demand Warning**
   - "You're approaching your peak demand record"
   - Helps avoid demand charges

3. **Efficiency Drop Alert**
   - "Your efficiency score dropped 20% this week"
   - Prompts investigation

---

### 5.3 Scheduled Reports
**Impact:** Medium | **Effort:** Low-Medium | **Timeline:** 2-3 hours

**Automated reports via email/notification:**

1. **Daily Summary**
   - Yesterday's total kWh and cost
   - Comparison to average
   - Top 3 consumers

2. **Weekly Report**
   - Weekly trend graph
   - Week-over-week comparison
   - Insights and recommendations

3. **Monthly Report**
   - Full monthly analysis
   - Cost breakdown by device
   - Efficiency metrics
   - Year-over-year comparison

---

## Priority 6: Long-Term Enhancements

### 6.1 Multi-Location Support
**Impact:** High (for some users) | **Effort:** Medium | **Timeline:** 4-6 hours

- Dashboard variables to switch between locations
- Comparison between multiple properties
- Aggregate statistics across portfolio

---

### 6.2 Mobile-Optimized Dashboard
**Impact:** Medium | **Effort:** Medium | **Timeline:** 4-6 hours

- Create separate mobile-friendly dashboard
- Key metrics only, vertical layout
- Large, touch-friendly controls

---

### 6.3 Energy Goals and Gamification
**Impact:** Medium | **Effort:** Medium | **Timeline:** 4-6 hours

**Features:**
- Set reduction goals (e.g., "Reduce usage by 10%")
- Progress tracking with visual indicators
- Achievement badges (e.g., "7 days under baseline")
- Leaderboards (for family members or community)

---

### 6.4 Historical Data Export
**Impact:** Low-Medium | **Effort:** Low | **Timeline:** 1-2 hours

**Features:**
- Export to CSV for external analysis
- Automated backups to cloud storage
- Data retention policy management

---

### 6.5 API Integration for External Services
**Impact:** Medium | **Effort:** High | **Timeline:** 8-12 hours

**Integrations:**
- Utility provider API (automatic bill import)
- Smart home automation triggers
- IFTTT/Zapier webhooks
- Voice assistant queries ("Alexa, what's my power usage?")

---

### 6.6 Carbon Footprint Tracking
**Impact:** Medium | **Effort:** Medium | **Timeline:** 3-4 hours

**Panel: "Carbon Emissions"**
- Converts kWh to CO₂ equivalent
- Based on grid mix (varies by region)
- Tracks monthly/annual emissions
- Equivalencies (e.g., "Equal to X miles driven")

**Formula:**
```
CO₂ (lbs) = kWh × grid_carbon_intensity
// US average: 0.92 lbs CO₂/kWh
// Varies significantly by state/country
```

---

## Implementation Roadmap

### Phase 1: Foundation (Weeks 1-2)
- ✅ Dashboard variables
- ✅ TOU rate support
- ✅ Baseline consumption panel
- ✅ Basic threshold alerts

### Phase 2: Analytics (Weeks 3-4)
- ✅ Comparative analysis panels
- ✅ Statistical summaries
- ✅ Cost breakdown by device
- ✅ Efficiency score

### Phase 3: Advanced Features (Weeks 5-8)
- ✅ Forecasting and predictions
- ✅ Load duration curve
- ✅ Anomaly detection
- ✅ Weather correlation

### Phase 4: Optimization (Weeks 9-12)
- ✅ Savings opportunities dashboard
- ✅ Predictive alerts
- ✅ Scheduled reports
- ✅ Energy goals tracking

### Phase 5: Long-Term (Ongoing)
- ✅ Multi-location support
- ✅ Mobile optimization
- ✅ Carbon footprint tracking
- ✅ External integrations

---

## Technical Requirements

### InfluxDB Considerations
- Ensure sufficient retention policy for historical analysis
- Consider downsampling for long-term data (e.g., 1-hour averages after 90 days)
- Monitor database size and performance

### Grafana Version
- Current: 7.4.0+
- Some features may require Grafana 9.0+ (unified alerting, canvas panels)
- Consider upgrade path

### External Data Sources
- Weather APIs: OpenWeatherMap, NOAA
- Utility APIs: Varies by provider
- Carbon intensity: ElectricityMap, WattTime

### Computational Load
- Forecasting and ML features require more CPU
- Consider dedicated processing for complex calculations
- Cache frequently accessed aggregations

---

## Estimated Costs

### Development Time
- **Priority 1 (Quick Wins):** 8-12 hours
- **Priority 2 (Visualizations):** 25-40 hours
- **Priority 3 (Cost Features):** 20-30 hours
- **Priority 4 (Analytics):** 30-50 hours
- **Priority 5 (Alerts):** 8-15 hours
- **Priority 6 (Long-term):** 25-40 hours
- **Total:** 116-187 hours (3-5 weeks full-time)

### External Services (Optional)
- Weather API: Free tier available, ~$0-50/month for premium
- Cloud storage backup: ~$5-20/month
- SMS alerts (Twilio): ~$0.01 per alert

### Infrastructure
- Increased InfluxDB storage: Varies by retention period
- Grafana Cloud (optional): Free tier available, Pro at $49/month

---

## Success Metrics

### Adoption Metrics
- Dashboard views per day
- Active alerts triggered
- Report subscriptions

### Value Metrics
- Energy consumption reduction (%)
- Cost savings ($)
- Time to identify issues (hours)
- User satisfaction score

### Technical Metrics
- Query performance (<2s response time)
- Dashboard load time (<3s)
- Alert accuracy (low false positives)
- System uptime (>99.9%)

---

## References and Resources

### Flux Query Language
- [InfluxDB Flux Documentation](https://docs.influxdata.com/flux/)
- [Flux Standard Library](https://docs.influxdata.com/flux/v0/stdlib/)

### Grafana Resources
- [Grafana Alerting](https://grafana.com/docs/grafana/latest/alerting/)
- [Dashboard Variables](https://grafana.com/docs/grafana/latest/dashboards/variables/)
- [Panel Plugins](https://grafana.com/grafana/plugins/)

### Energy Monitoring Best Practices
- [Home Energy Monitoring Guide (Energy.gov)](https://www.energy.gov/)
- [Time-of-Use Rate Optimization](https://www.energy.gov/energysaver/using-electricity-wisely)

### Machine Learning for Energy
- [NILM Toolkit](https://github.com/nilmtk/nilmtk)
- [Energy Disaggregation Research](https://arxiv.org/abs/1507.06594)

---

## Conclusion

This improvement plan provides a comprehensive roadmap for enhancing the home energy monitoring dashboard. Priority 1 items offer quick wins with minimal effort, while later priorities add sophisticated analytics and optimization features. The modular approach allows for incremental implementation based on user needs and available development time.

**Next Steps:**
1. Review and prioritize features based on user requirements
2. Set up development environment for testing new panels
3. Begin with Priority 1 quick wins for immediate value
4. Gather user feedback and iterate on implementations
5. Document all new features in updated guides

**Maintenance Considerations:**
- Regular review of alert thresholds
- Update electricity rates as they change
- Monitor dashboard performance as data grows
- Keep Grafana and InfluxDB updated
- Backup configurations and queries

---

**Document Version:** 1.0
**Last Updated:** 2025-11-10
**Author:** Energy Dashboard Enhancement Plan
**Contact:** See README.md for support information
