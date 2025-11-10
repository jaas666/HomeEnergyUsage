# Home Energy Usage Dashboard (InfluxDB Edition)

Monitor your home energy consumption with this Grafana dashboard using InfluxDB v2 and Flux queries.

## Hardware Required

- **Emporia Vue 2** - Energy monitoring device (~$100)
- **Server/VM** - Running HomeAssistant & InfluxDB

## Architecture

```
Emporia Vue → HomeAssistant → InfluxDB v2 → Grafana
```

**Data Flow:**
1. Emporia Vue monitors your circuit breaker and sends data to HomeAssistant
2. HomeAssistant processes sensor data and writes to InfluxDB
3. Grafana queries InfluxDB using Flux and displays the dashboard

## Prerequisites

- HomeAssistant installed and running
- InfluxDB v2.x installed and running
- Grafana installed (or Grafana Cloud account)
- Emporia Vue 2 device installed on your circuit breaker

## Setup Instructions

### Step 1: Install Emporia Vue Hardware

Install the Emporia Vue 2 on your circuit breaker panel. Follow the instructions in the Emporia mobile app. The app provides safety guidelines and step-by-step installation instructions.

**Safety Warning:** Working with electrical panels can be dangerous. If you're not comfortable, hire a licensed electrician.

### Step 2: Connect Emporia Vue to HomeAssistant

1. Open the Emporia Vue mobile app and set up your device
2. In HomeAssistant, go to **Settings → Devices & Services → Add Integration**
3. Search for "Emporia Vue"
4. Log in with your Emporia Vue credentials
5. Your circuits should now appear as sensors in HomeAssistant

### Step 3: Configure InfluxDB Integration in HomeAssistant

Add the following to your HomeAssistant `configuration.yaml`:

```yaml
influxdb:
  api_version: 2
  host: YOUR_INFLUXDB_HOST          # e.g., localhost or 192.168.1.100
  port: 8086
  token: YOUR_INFLUXDB_TOKEN        # Generate in InfluxDB UI
  organization: Home                # Your InfluxDB org
  bucket: HomeAssistant             # Your InfluxDB bucket
  ssl: false                        # Set to true if using HTTPS
  verify_ssl: false
  tags:
    source: homeassistant
  tags_attributes:
    - friendly_name
    - device_class
  default_measurement: state
  include:
    entities:
      - sensor.power_*              # Include all Emporia power sensors
      - sensor.*_power_*            # Adjust based on your entity names
```

**Important:** Replace the placeholders with your actual InfluxDB connection details.

**To get your InfluxDB token:**
1. Open InfluxDB UI (usually http://your-server:8086)
2. Go to **Data → Tokens**
3. Generate a new token with write access to your HomeAssistant bucket

### Step 4: Restart HomeAssistant

Restart HomeAssistant to activate the InfluxDB integration:
- Settings → System → Restart

### Step 5: Verify Data is Flowing to InfluxDB

1. Open InfluxDB UI
2. Go to **Data Explorer**
3. Select bucket: `HomeAssistant`
4. You should see measurements like `W`, `kWh`, or `state`
5. Filter by your Emporia sensors to confirm data is being written

### Step 6: Import Grafana Dashboard

#### Option A: Using Grafana UI

1. Open Grafana
2. Go to **Configuration → Data Sources**
3. Add **InfluxDB** data source:
   - Query Language: **Flux**
   - URL: `http://your-influxdb-host:8086`
   - Organization: `Home`
   - Token: Your InfluxDB token
   - Default Bucket: `HomeAssistant`
4. Click **Save & Test**
5. Go to **Dashboards → Import**
6. Upload `home-energy-usage-influxdb.json`
7. Select your InfluxDB data source
8. Click **Import**

#### Option B: Using Grafana Cloud

1. Log in to Grafana Cloud
2. Add InfluxDB data source with your connection details
3. Import the dashboard JSON file

### Step 7: Customize Dashboard

You may need to customize the dashboard based on your sensor names:

1. **Find your sensor naming convention:**
   - Open InfluxDB Data Explorer
   - Look at the `friendly_name` tag values
   - Note the naming pattern (e.g., "Total Power 1Mon", "Total Power 1D")

2. **Update dashboard queries if needed:**
   - Edit panels that don't show data
   - Modify `friendly_name` filters to match your sensors
   - See `INFLUXDB_QUERY_GUIDE.md` for details

3. **Adjust electricity rates:**
   - Edit "Current Month Bill" panel
   - Modify the calculation: `r._value * 0.0965` (change 0.0965 to your rate per kWh)
   - Modify the formula to match your utility's billing structure

## Dashboard Panels

The dashboard includes:

1. **Top 5 Devices Currently Running** - Real-time power consumption by device
2. **Current Month Total Power Usage** - kWh consumed this month
3. **Current Month Bill** - Estimated electricity bill (customize rate)
4. **Current Day Power Total** - Today's kWh consumption
5. **Current Day Power Cost** - Today's estimated cost
6. **Month Power Total** - Bar gauge showing monthly usage by device
7. **Hourly Power by Device** - Stacked bar chart of hourly consumption
8. **Top 5 Devices Last Week** - Pie chart of biggest energy consumers
9. **Last 7 Days Power Usage** - Daily consumption trend
10. **Power Spikes This Week** - Rate of change to identify unusual usage

## Troubleshooting

### No Data in Dashboard

1. **Check HomeAssistant logs:**
   ```
   Settings → System → Logs
   Search for "influxdb"
   ```

2. **Verify InfluxDB connection:**
   - Check that InfluxDB is running
   - Verify token has write permissions
   - Check network connectivity

3. **Check sensor names:**
   - In HomeAssistant, go to Developer Tools → States
   - Search for your Emporia sensors
   - Note the exact entity IDs and friendly names
   - Update dashboard queries if they don't match

### Wrong Measurements

1. **Check InfluxDB schema:**
   ```flux
   import "influxdata/influxdb/schema"

   schema.measurements(bucket: "HomeAssistant")
   ```

2. **Update queries to match your measurements:**
   - Replace `W` with your wattage measurement name
   - Replace `kWh` with your energy measurement name
   - See `INFLUXDB_QUERY_GUIDE.md` for examples

### Incorrect Calculations

- Verify your electricity rate in the cost calculation panels
- Check that sensor values are in expected units (W vs kW, Wh vs kWh)

## Customization

### Electricity Rate

Default rate: **$0.0965/kWh**

To change:
1. Edit "Current Month Bill" panel
2. Find the query: `map(fn: (r) => ({ r with _value: (r._value * 0.0965) * 1.05 + 14.0 }))`
3. Modify:
   - `0.0965` - Rate per kWh
   - `1.05` - Tax/fees multiplier (5% in this example)
   - `14.0` - Fixed monthly fee

### Adding More Panels

Use the existing queries as templates. See `INFLUXDB_QUERY_GUIDE.md` for Flux query examples.

## Files in this Repository

- `home-energy-usage-influxdb.json` - Grafana dashboard (InfluxDB/Flux version)
- `home-energy-usage_rev1.json` - Original Prometheus dashboard (legacy)
- `configuration.yml` - Example HomeAssistant configuration snippet
- `README.md` - This file
- `INFLUXDB_QUERY_GUIDE.md` - Flux query customization guide

## Credits

Original Prometheus version concept adapted for InfluxDB v2 with Flux queries.

## License

MIT License - Feel free to use and modify!

## Support

For issues related to:
- **Emporia Vue**: Check [Emporia Support](https://www.emporiaenergy.com/support)
- **HomeAssistant**: Visit [HomeAssistant Community](https://community.home-assistant.io/)
- **InfluxDB**: See [InfluxDB Documentation](https://docs.influxdata.com/influxdb/v2/)
- **Grafana**: Check [Grafana Documentation](https://grafana.com/docs/)
