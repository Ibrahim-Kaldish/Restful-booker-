# Restful-Booker API Performance Test — JMeter

Performance test suite for the [Restful-Booker API](https://restful-booker.herokuapp.com), built with Apache JMeter. Simulates concurrent users authenticating and performing full CRUD operations on bookings, with weighted traffic distribution, parameterized test data, and response validation.

## Project Structure

```
NTI-Assignment/
├── herokuapp.jmx              # Main JMeter test plan
├── Data/
│   ├── login_credentials.csv  # username, password
│   └── booking_guests.csv     # firstname, lastname
└── README.md
```

## Test Plan Overview

| Element | Details |
|---|---|
| Target host | `restful-booker.herokuapp.com` (HTTPS) |
| Threads (users) | 1000 |
| Ramp-up time | 1 second |
| Loop count | 1 |
| Error handling | Continue on sampler error |

### Flow

1. **Login** (`POST /auth`) — authenticates using credentials from `login_credentials.csv`; auth token extracted via JSON Extractor (`$.token`) and injected into the `Cookie` header (`token=${token}`) for all subsequent requests.
2. **Throughput Controller — CreatingBookings (70%)**
   - `POST /booking` — creates a booking using guest data from `booking_guests.csv`; `bookingid` extracted via JSON Extractor.
   - `GET /booking/${booking_id}` — retrieves the created booking.
   - `GET /booking` — retrieves all booking IDs.
3. **Throughput Controller — UpdatingBookings (15%)**
   - `PUT /booking/${booking_id}` — full update of booking fields.
   - `PATCH /booking/${booking_id}` — partial update (firstname/lastname).
4. **Throughput Controller — DeletingBookings (15%)**
   - `DELETE /booking/${booking_id}` — removes the booking.

### Assertions & Timers

- **Response Assertion** — expects HTTP `200` on responses.
- **Duration Assertion** — fails any request exceeding `2000 ms`.
- **Gaussian Random Timer** — 300 ms delay ± 100 ms deviation between requests, to simulate realistic user think-time.

### Listeners

- View Results Tree
- Summary Report
- Aggregate Report
- Graph Results

## Prerequisites

- [Apache JMeter](https://jmeter.apache.org/download_jmeter.cgi) 5.6.3 or later
- Java JDK 8+ installed and on `PATH`

## Running the Test

### GUI mode (for debugging / recording)

```bash
jmeter -t herokuapp.jmx
```

> GUI mode is for test development only — never use it for actual load generation.

### CLI / non-GUI mode (for actual load testing)

```bash
jmeter -n -t herokuapp.jmx -l results.jtl -e -o report/
```

- `-n` — non-GUI mode
- `-l results.jtl` — raw results log
- `-e -o report/` — generates an HTML dashboard report in `report/`

## Test Data

Update the CSV files under `Data/` with valid values before running:

**login_credentials.csv**
```
username,password
admin,password123
```

**booking_guests.csv**
```
firstname,lastname
John,Doe
Jane,Smith
```

Both CSV Data Set Configs are set to `recycle = true`, so values loop automatically once all rows are consumed.

## Proxy Recording Setup

The test plan includes an **HTTP(S) Test Script Recorder** (`ProxyControl`) on port `8888`, used to record new requests from a browser directly into the test plan.

### Steps to record

1. **Configure JMeter's proxy**
   - Open the test plan in the JMeter GUI.
   - Under the recorder element, confirm the port is `8888` (change if already in use).
   - Add a **Target Controller** (where recorded samplers will be added) and an entry in **URL Patterns to Include/Exclude** if you want to filter static assets (`.css`, `.js`, images, etc.).
   - Click **Start** on the recorder.

2. **Configure your browser to use the proxy**
   - Set the browser's (or OS-level) proxy to `localhost:8888`.
   - Or use a dedicated proxy-switching browser extension (e.g. FoxyProxy) to avoid affecting other traffic.

3. **Browse the target application**
   - Navigate through the app as a normal user; each request is captured and added under the Target Controller as it happens.

4. **Stop the recorder** once done, and clean up the recorded samplers (remove redundant headers, add correlation/extraction where needed, parameterize hardcoded values).

## HTTPS Certificate Handling

Since Restful-Booker is served over HTTPS, recording HTTPS traffic requires JMeter to act as a man-in-the-middle: it dynamically generates and signs certificates for each domain using its own root CA, so the browser must trust that CA first.

### Steps

1. **Locate JMeter's root CA certificate**
   - On first proxy start, JMeter generates `ApacheJMeterTemporaryRootCA.crt` (or `ApacheJMeterTemporaryRootCACert.crt` depending on version) inside the JMeter `bin/` directory.

2. **Install the certificate in your browser/OS trust store**
   - **Chrome/Edge (Windows):** `Settings → Privacy and security → Security → Manage certificates → Trusted Root Certification Authorities → Import` → select the `.crt` file.
   - **Firefox:** Firefox uses its own certificate store — `Settings → Privacy & Security → Certificates → View Certificates → Authorities → Import`.
   - **macOS:** Open the `.crt` file with **Keychain Access**, add it to the `System` or `login` keychain, then double-click the entry and set **Trust → When using this certificate → Always Trust**.
   - **Linux:** Depends on distro; typically copy the `.crt` into `/usr/local/share/ca-certificates/` and run `sudo update-ca-certificates`.

3. **Regenerate the CA if needed**
   - Delete the existing cert files in JMeter's `bin/` folder and restart the recorder to force JMeter to generate a fresh CA (useful if the previous one expired or was corrupted).
   - By default, JMeter-generated certs are valid for **7 days** — configurable via `proxy.cert.validity` in `jmeter.properties` (in days) if longer-lived recording sessions are needed.

4. **Verify**
   - With the browser proxy pointed at JMeter and the CA trusted, browsing to `https://restful-booker.herokuapp.com` should show no certificate warnings, and requests should appear in the recorder.

> Never leave the browser proxy pointed at JMeter outside of active recording sessions — all traffic (including sensitive sites) will be intercepted while it's active.

## Results & Reporting

After a non-GUI run, open `report/index.html` in a browser for the full HTML dashboard (response time percentiles, throughput, error rate breakdowns, etc.).

## Notes

- `ThreadGroup.num_threads = 1000` with `ramp_time = 1` second is an aggressive ramp — adjust for realistic load testing to avoid overwhelming the target or triggering rate limiting on the free Heroku-hosted API.
- The `DeleteBooking` sampler is currently configured with method `GET` instead of `DELETE` — verify before relying on this section to actually delete bookings.
