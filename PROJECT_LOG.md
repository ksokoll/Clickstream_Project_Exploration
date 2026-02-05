# Project Log: Clickstream Analytics & Anomaly Detection

**Project Type:** End-to-End Data Engineering & Analytics  
**Duration:** January 22. February 4, 2026 (2 weeks)  
**Status:** Completed

---

## Project Overview

I built a complete clickstream analytics system to diagnose a 35% conversion rate drop for an e-commerce retailer. Implemented streaming data pipeline, statistical anomaly detection, and identified Safari iOS browser bug causing checkout failures.

**Key Achievement:** System would have detected the bug in 3 days vs 2 weeks without automated monitoring.

---

## Phase 0: Planning & Data Preparation
January 22-26, 2026

### The Beginning

I started with a clear business problem: an electronics retailer experiencing an unexplained 35% drop in conversion rates. Rather than building yet another "predictive maintenance" portfolio project, I wanted something with direct business impact and realistic complexity. Clickstream analytics offered exactly that. its what real companies use to understand their customers, and its technically challenging enough to demonstrate end-to-end data engineering skills.

The first major decision was choosing the right dataset. I settled on the Retail Rocket Recommender System dataset from Kaggle. 2,7 million events covering 4,5 months of real e-commerce activity. It wasnt too large to be unmanageable, but substantial enough to reveal realistic patterns. The dataset included the three critical event types I needed: views (browsing), addtocart (intent), and transactions (conversion).

### Injecting Realistic Problems

Heres where things got interesting. Since I want to simulate a real business case with the electronics retailer, I needed to simulate the Safari iOS bug convincingly. Simply deleting transactions would be obvious and unrealistic. Instead, I thought about what actually happens when a checkout fails. users add items to cart but cant complete the purchase. So I developed a contamination strategy: take 80% of Safari iOS transactions from the last two weeks and convert them to addtocart events. Why 80% instead of 100%? Because real bugs rarely affect every single user. some have cached versions, different browser versions, or simply get lucky with timing.

The technical implementation required careful thought. I couldnt just randomly assign devices and browsers to users. that would create impossible combinations like Safari on Android or Edge on macOS. I built a conditional distribution system where browser choices depended on operating systems, and both depended on device types. Every event from a single visitor had to maintain consistency. if visitor_12345 started on iOS Safari, all their events stayed on iOS Safari.

Performance became an immediate concern when processing 1.4 million unique visitors. My initial approach using Python loops would have taken hours. I pivoted to vectorized NumPy operations, creating mapping tables for all visitors at once and merging them back to the main dataset. The speedup was dramatic. from potentially hours down to minutes.

After contaminating the data, I validated thoroughly. The numbers told the story: 572 transactions in the affected period, 458 of them (80%) converted to addtocart events. Transaction count dropped from 16,235 to 15,777 (-458), while addtocart increased from 50,550 to 51,008 (+458). The math checked out perfectly. I exported the contaminated dataset as CSV, ready for the next phase.

**Key Decision. Transparent Synthetic Data:** I decided early on to be completely transparent about the bug injection. This wasnt about tricking anyone into thinking I found a "real" bug. it was about demonstrating systematic troubleshooting skills. The README would clearly state: "Synthetic bug injected for demonstration purposes." This honesty actually strengthens the portfolio because it shows I understand the difference between demo projects and production scenarios.

---

## Phase 1: Building the Streaming Pipeline
**Duration:** January 27, 2026

### The Microsoft Fabric Pivot

My original plan was elegant: use Microsoft Fabrics Event Streams to build a modern streaming pipeline. Id connect to Azure, set up the event hub, and stream the CSV data like real GA4 events flowing from an e-commerce site. The narrative was perfect. a company upgrading from batch CSV exports to real-time streaming analytics.

Then reality hit. I logged into Azure to find my account locked behind MFA that I couldnt bypass. Hours of troubleshooting various authentication methods led nowhere. I was completely blocked from Fabric.

This was a moment where I could have given up or spent days fighting Azure support. Instead, I made a strategic pivot: Apache Kafka running locally in Docker. Initially, this felt like a downgrade. moving from a managed cloud service to local infrastructure. But as I thought it through, I realized this was actually better for the portfolio. Running Kafka myself meant understanding topics, partitions, consumer groups, and offset management at a deeper level. Plus, its completely reproducible. anyone can `docker-compose up` and see the whole system running, no cloud account required.

### Docker Compose and the Kafka Stack

I set up Docker Compose with three services: Zookeeper (for Kafka coordination), Kafka itself, and TimescaleDB (for time-series storage). The containers pulled down about 850MB of images, but once running, the system was rock solid. I created a `clickstream-events` topic and verified it with Kafkas command-line tools.

Next came the Producer. I rewrote what would have been Azure Event Hubs code to use the `kafka-python` library instead. The Producer would read the CSV line by line, convert each row to JSON, and stream it to Kafka at a realistic rate. 1000 events per second to simulate actual e-commerce traffic. Rate limiting was crucial here; without it, Id blast all 2.7 million events instantly, which wouldnt demonstrate anything about handling real-time data flow.

Then I hit my first major bug. The Producer logs showed events being processed: "Sending event 1... Sending event 2..." But when I ran a Consumer, nothing appeared. I spent an hour checking Kafka topics, verifying configurations, even restarting containers. Finally, I found it. I was printing the events but never actually calling `producer.send()`. Classic mistake: the code logged the *intent* to send but never executed the actual send operation. Adding `producer.send(topic, value)` and `producer.flush()` fixed it immediately.

I built a simple Consumer that would read from Kafka and print events to the terminal. No database integration yet. I wanted to validate the Kafka connection first before adding another layer of complexity. When I saw events flowing from Producer to Kafka to Consumer, all displaying correctly in my terminal, that was the first major victory.

---

## Phase 1 (Session 2): The 4-Hour Container Networking Battle
**Duration:** January 27, 2026 (4 hours)

### Moving the Consumer to Docker

With Kafka working, I needed to write events to TimescaleDB. Thats when Windows encoding issues reared their ugly head. The `psycopg2` library, Pythons PostgreSQL adapter, was throwing Unicode errors when connecting from my Windows machine. Id seen this problem before. Windows handles Unicode differently than Linux, and PostgreSQL drivers can be finicky about character encoding.

The solution was moving the Consumer into a Docker container running Linux. I created a new service in Docker Compose, packaged my Consumer code, and configured it to run alongside Kafka and TimescaleDB. This should have been straightforward, but container networking had other plans.

The Consumer kept failing with "Connection refused" errors. It was trying to connect to `localhost:9092`, which made sense on my Windows host but was completely wrong inside a Docker container. In Dockers internal network, containers dont use `localhost`. they use container names as hostnames. I needed to change the Kafka bootstrap server from `localhost:9092` to `kafka:29092`.

But why 29092 and not 9092? This is where Kafkas listener configuration gets interesting. Kafka has `ADVERTISED_LISTENERS` with two separate endpoints: one for external connections from the host machine (localhost:9092) and one for internal container-to-container communication (kafka:29092). The Producer running on my Windows host uses localhost:9092, while the Consumer running inside Docker uses kafka:29092. Same Kafka instance, different network routes.

### The Mystery of the Silent Logs

I got the Consumer running but hit another frustrating issue. no output. The container started successfully according to `docker ps`, but when I checked logs with `docker logs consumer`, nothing appeared. No error messages, no event processing logs, nothing. The Consumer was running but completely silent.

After significant debugging, I discovered Pythons output buffering. In interactive mode (running Python directly in a terminal), print statements appear immediately. In non-interactive mode (like inside a Docker container), Python buffers output for efficiency. This means print statements might not appear in logs for seconds or even minutes, making debugging nearly impossible.

The fix required two changes: setting the `PYTHONUNBUFFERED=1` environment variable in Docker Compose and using the `python -u` flag in my containers command. Both tell Python to flush output immediately. Suddenly, logs appeared in real-time, and I could watch events flowing through the system.

### Consumer Groups and Offset Management

With logging working, I noticed something odd. the Consumer was re-processing events from the beginning every time I restarted it, even though Kafka should remember where each consumer left off. This led me down the rabbit hole of Consumer Groups and offset management.

Kafka tracks which messages each consumer group has already processed using offsets. essentially bookmarks for each partition. The `auto_offset_reset=earliest` setting only applies when Kafka doesnt have any stored offset for a consumer group. Once offsets exist, Kafka uses them, which is why the Consumer wasnt reading from the beginning after the first run.

For testing, I needed a way to reset and reread all events. The solution was Kafkas consumer group management commands: `kafka-consumer-groups --reset-offsets --to-earliest`. This cleared the offset history and forced the Consumer to start from the beginning again.

### Success: End-to-End Pipeline

After four hours of debugging networkin, buffering, and offset issues, I finally had it working: CSV file → Producer on Windows → Kafka → Consumer in Docker → TimescaleDB. I watched 425 events flow through the entire pipeline and verified them in the database with a simple SQL query. The system was complete.

Looking back, this session taught me the most about production systems. The initial Kafka setup took 30 minutes. The debugging took four hours. Thats typical in data engineering. getting individual components running is straightforward; making them work together reliably is where the real engineering happens.

---

## Phase 2: The Great Parquet Pivot
**Duration:** January 28, 2026

### Seven Hours of ODBC Hell

With data flowing into TimescaleDB, the next step seemed simple: connect Power BI and build visualizations. I installed the PostgreSQL ODBC driver, configured the connection with the correct hostname, database name, username, and password. Power BIs connection dialog opened, everything looked right. I clicked "Connect."

`Password authentication failed for user clickstream`

Thats odd, I thought. I triple-checked the password, tried again. Same error. Maybe its a permission issue? I granted all privileges to the clickstream user in PostgreSQL. Still failed. Perhaps Power BI doesnt like custom users? I created a `postgres` superuser with a simple `admin` password. Failed again.

I dove into PostgreSQLs authentication configuration file (`pg_hba.conf`). The file controls how PostgreSQL authenticates different types of connections. I saw rules requiring `scram-sha-256` authentication. a secure but picky authentication method. I changed it to `md5` (less secure but more compatible), reloaded PostgreSQL configuration. Still failed.

What really confused me was the error message variation. Sometimes it said `user clickstream`, sometimes `user postgre` (note the missing s). Wait. why would it truncate postgres to postgre? Thats not a database issue, thats a client-side encoding problem.

I tried changing `localhost` to `127.0.0.1` thinking DNS resolution might be the issue. Nope. I cleared Power BIs data source cache. Nothing. I reinstalled the ODBC driver. Still broken. After seven hours of this. checking authentication methods, reading PostgreSQL logs, trying different connection strings, searching forums for similar issues. I realized I was fighting a Windows-specific ODBC encoding bug that had no clear solution.

### The Pragmatic Decision

This is where I had to be honest with myself. The goal of this project wasnt "demonstrate mastery of PostgreSQL ODBC drivers on Windows." The goal was "build a clickstream analytics system that detects conversion anomalies." I was seven hours deep in troubleshooting a tool integration issue that had nothing to do with the core technical skills I wanted to demonstrate.

I asked myself: what if I just used Parquet files instead? Power BI has native Parquet support. Parquet is the production standard for data lakes. Databricks uses it, Snowflake uses it, Delta Lake is built on it. File size would be dramatically better than CSV (columnar compression). No authentication complexity. And honestly, nobody in an interview would ask "why didnt you use PostgreSQL?" because Parquet is perfectly legitimate for this use case.

Phew! The pivot took less than 30minutes. I modified the Consumer to batch events and write Parquet files instead of database rows. Actually, I ended up writing a simpler script that just converted the CSV directly to Parquet files. 10,000 events per file to mimic streaming behavior. Total result: 270 Parquet files, roughly 150MB (compared to 1GB for the CSV).

### Consumer Stability Issues

Before settling on direct CSV-to-Parquet conversion, I tried getting the Consumer to write Parquet files incrementally as events streamed in. It worked beautifully for the first 130,000 events, then mysteriously stopped. Logs showed "Listening for events..." but no new files were created. Kafka reported LAG = 0, meaning the Consumer thought it was caught up, but 1.35 million events remained in the topic.

I tried everything: changed consumer groups (v1 → v2 → v3), reduced batch size from 1000 to 500 events, added explicit `consumer.commit()` calls, reduced Kafkas `max_poll_records` setting, added comprehensive error handling with try/except blocks. Some attempts got further. 660,000 events before hanging. but nothing was reliable.

At this point, I made another pragmatic call. Debugging Kafka consumer stability is a separate problem from demonstrating streaming architecture. The portfolio already showed Producer → Kafka → Consumer flow. Whether the Consumer writes to Parquet incrementally or I convert CSV to Parquet in one batch doesnt change the fundamental architecture demonstration. The CSV conversion took two minutes versus potentially hours more debugging.

### Power BI Finally Working

Loading 270 Parquet files into Power BI Desktop was trivial. used the Folder connector, clicked "Combine Files," and Power BI automatically merged everything into a single table. I built initial visualizations: event distribution over time, browser/OS/device breakdowns, and a date slicer to filter specific periods.

The foundation was in place. Now I could actually analyze the data and hunt for that Safari iOS bug.

---

## Phase 3: Finding the Bug Through Statistical Analysis
**Duration:** January 28, 2026 (Session 3)

### The Detective Work

With Power BI dashboard built, I began systematic analysis. First, the overall picture: 2.7 million events across 142 days. Event distribution showed the expected e-commerce funnel. 96.66% views (browsing), 2.5% addtocart (intent), and just 0.82% transactions (conversion). Browser split was Chrome at 51%, Safari 40%, with Firefox and Edge trailing at 4-5% each.

The retailers complaint was about the last two weeks specifically, so I created a "normal period" (everything before) and "anomaly period" (last two weeks) comparison. Average daily events dropped from 14,390 to 12,450. That seemed significant until I calculated standard deviation. 4,440 events. The drop was within one standard deviation (normal range: 9,950. 18,830 events/day), meaning it could just be regular traffic variance. Traffic wasnt the problem.

### Building DAX Measures for Statistical Analysis

I needed a more rigorous approach than just eyeballing charts. I built DAX measures to calculate means and standard deviations across different dimensions. The pattern became my template: group all events by date, count events per day, calculate standard deviation across those daily counts. This gave me a single number representing normal daily variance.

For transactions specifically, daily average was 113, dropping to 77 in the anomaly period. Again, this was within ±1σ (range: 54-172 transactions/day), so normal variance couldnt explain everything. But when I split by device, something popped: Mobile transactions averaged 51.6/day normally but only 34.4/day in the anomaly period. That was below -1σ. Desktop and Tablet were both within normal ranges.

### The iOS Smoking Gun

Device level pointed to Mobile, so I sliced by operating system. Android, Linux, macOS, Windows. all within normal ranges. But iOS? Normal period: 33.5 transactions/day. Anomaly period: 10.25 transactions/day. Thats not just below -1σ, its below -4σ. In statistical terms, thats screaming "SOMETHING IS VERY WRONG HERE."

For completeness, I checked browsers. Chrome, Edge, Firefox all looked normal. Safari was at 19.9 transactions/day in the anomaly period versus 39.9 normally. technically within ±1σ but right at the edge of the normal range. Worth investigating.

### The Combination Analysis

The pattern was clear: Mobile + iOS + Safari. I filtered the dashboard to just that combination and compared event types:

**Normal Period:**
- Views: 6,940/day
- AddToCart: 177.65/day  
- Transactions: 51.57/day

**Anomaly Period:**
- Views: 8,190/day (actually up 18%!)
- AddToCart: 242.92/day (up 37%!)
- Transactions: 34.36/day (down 33%)

This was the smoking gun as Views were increasing and the Safari iOS users were definitely browsing the site. AddToCart events surged, they were trying to buy things but transactions crashed. the Users were getting all the way to checkout and then... something broke. They couldnt complete purchases.

### The Conversion Rate Proof

To quantify the impact, I calculated conversion rates (transactions divided by views):

- Normal: 51.57 / 6,940 × 100 = **0.74%**
- Anomaly: 34.36 / 8,190 × 100 = **0.42%**

A 43 percentage point drop in conversion rate. Not 43% worse. the conversion rate literally dropped by 0.32 percentage points, which represented a 43% decline from baseline. For an e-commerce site, this is catastrophic.

The evidence was overwhelming: Safari iOS users hit a checkout bug that prevented transaction completion. The transactions were being misclassified as addtocart events. exactly what my data contamination strategy had simulated.

---

## Phase 4: Anomaly Detection System
**Duration:** January 28-31, 2026

### From Manual Analysis to Automated Detection

Finding the Safari iOS bug through Power BI took hours of systematic slicing and statistical analysis. In production, you cant manually check dashboards every day hoping to notice problems :) I needed an automated system that would detect anomalies the moment they appeared and alert stakeholders before significant revenue was lost.

The goal: build a system that would have caught this bug in 3 days instead of 2 weeks. The approach needed to be statistically rigorous (not just "this looks weird"), handle normal variance (weekends vs weekdays), and minimize false positives (dont alert for every minor fluctuation).

### The Static Analysis Script

I started with a retrospective analysis script. run it once over all 107 days of Safari iOS data and see what it finds. The algorithm needed three key components:

**1. Baseline Calculation:** For any given day, whats "normal"? I chose a same-weekday approach. compare each Saturday against the last 4 Saturdays, each Monday against the last 4 Mondays, etc. This automatically accounts for weekly patterns in e-commerce (Friday shopping peaks, Sunday lulls) without needing a holiday calendar or complex seasonality modeling.

**2. Threshold Detection:** When is something anomalous? I used ±2σ (two standard deviations), which in normal distribution covers 95% of data. Values outside that range are statistically unlikely to be random variance. This is standard practice in statistical process control. not too sensitive (±1σ would trigger constantly), not too conservative (±3σ would miss real issues).

**3. Consecutive Day Rule:** A single anomalous day could be a fluke. server downtime, deployment issues, random variance. But three consecutive anomalous days? Thats a systematic problem requiring immediate investigation. This rule distinguishes between "monitor this" and "wake someone up now."

### The Baseline Contamination Problem

My first implementation used mean and standard deviation for the baseline. When I ran it on September 11th (well into the bug period), it should have screamed "ANOMALY!" Instead, it said "looks fine." The conversion rate was 0.22%, which was terrible, but the threshold was -0.35% (negative!). How could the threshold be negative?

The problem was baseline contamination. The "last 4 weeks" lookback window included bug days. If the last 4 Saturdays had conversion rates of [0.8%, 0.75%, 0.8%, 0.1%], the mean was 0.6125%. pulled down by that 0.1% bug value. The standard deviation was huge (0.49) because of the variance between normal and bug days. Threshold = 0.6125. (2 × 0.49) = -0.37%. The system thought conversion rates below zero were normal!

### Pivoting to Robust Statistics

I needed statistics that wouldnt be fooled by outliers. The answer was replacing mean with median and standard deviation with MAD (Median Absolute Deviation). 

**Median for baseline:** With the same [0.8%, 0.75%, 0.8%, 0.1%] example, the median is 0.775%. it ignores the outlier. The median is the middle value when sorted, so even if one value is drastically wrong, it doesnt affect the result.

**MAD for sigma:** Instead of squaring deviations (which makes outliers HUGE), MAD takes the median of absolute deviations. For our example: `median([|0.8-0.775|, |0.75-0.775|, |0.8-0.775|, |0.1-0.775|]) = median([0.025, 0.025, 0.025, 0.675]) = 0.025`. The outlier (0.675) is ignored!

**Why 1.4826?** MAD and standard deviation measure spread differently. To make them comparable (so I can still use ±2σ terminology), theres a scaling factor: `sigma = MAD × 1.4826`. This number comes from the mathematical relationship between MAD and standard deviation in a normal distribution. It lets me say "±2σ threshold" even though Im using MAD under the hood.

After switching to median + MAD, the system worked perfectly. September 11th was correctly flagged as anomalous (conversion 0.22% vs baseline 0.84%, deviation -6.4σ). The threshold stayed realistic at 0.67% instead of going negative.

### Results: Finding the Bug Automatically

Running the static analysis over all 107 days of Safari iOS data:

- **Total Days:** 107
- **Anomalous Days:** 16 (15%)
- **Critical Periods (≥3 consecutive days):** 2
 . Period 1: September 1-12 (12 days)
 . Period 2: September 16-18 (3 days)

The most extreme deviations were -8.2σ (Sept 4), -6.8σ (Sept 14), and -6.5σ (Sept 10). For context, in normal distribution, -3σ occurs about 0.1% of the time. -8σ is astronomically unlikely to be random chance.

**The key finding:** The system would have triggered a critical alert on September 3rd. just 3 days after the bug started. In reality, it took business stakeholders 2 weeks to notice the problem through manual dashboard review. Thats 79% faster detection, which translates directly to reduced revenue loss.

### Multi-Combination Validation

To ensure Safari iOS was truly the isolated problem (and not a false positive in my detection algorithm), I ran the same analysis on 4 other browser/OS/device combinations:

**Safari macOS Desktop:** 16 anomalous days (15%) but zero critical periods. The anomalies were scattered single days, not consecutive streaks, indicating normal variance rather than systematic issues.

**Chrome Android Mobile:** 6 anomalous days (5.6%), zero critical periods. Very stable.

**Chrome Windows Desktop:** 7 anomalous days (6.5%), zero critical periods. Also stable.

**Firefox Windows Desktop:** 16 anomalous days (15%), zero critical periods. Similar pattern to Safari macOS. some variance but no systematic problem.

Safari iOS was the **only** combination with 3+ consecutive anomalous days. This proved the bug was platform-specific (iOS), not browser-wide (Safari on macOS was fine) or device-wide (Mobile Chrome was fine). The multi-combination analysis transformed directly potinted to the root cause.

### The Production Architecture: OOP Refactoring

The static analysis script proved the concept, but it was 500+ lines of procedural code. functions calling functions in main(). For a portfolio project demonstrating production readiness, I needed proper software architecture.

I refactored into a modular OOP design:

**config.py**. Centralized settings (sigma threshold, baseline weeks). In production, these would be environment variables or config files that can be changed without touching code.

**models.py**. Pydantic models for type safety. Each data structure (DailyConversion, BaselineMetrics, AnomalyResult, ConsecutiveStreak, DailyCheckResult) is defined with explicit types. This prevents runtime errors. if you try to pass a string where a float is expected, Pydantic catches it immediately with a clear error message.

**data_loader.py**. Handles Parquet file loading with methods for different scenarios (load all, load before date, load for specific date). Separates data access from business logic.

**processors/** directory with three specialized classes:
- **ConversionCalculator**. Computes daily conversion rates for specific browser/OS/device combinations
- **BaselineCalculator**. Implements the median + MAD baseline logic with same-weekday lookback
- **AnomalyDetector**. Applies ±2σ threshold and detects consecutive day streaks

**pipeline.py**. Orchestrates the workflow. Takes a target date, loads historical data, calculates baseline, loads new events for that date, detects anomalies, checks for consecutive streaks, and returns a structured result. Single responsibility: coordinate components.

**main.py**. CLI entry point. Parses command-line arguments, calls the pipeline, formats output for humans. Separated from pipeline.py so the pipeline can be imported and used programmatically (e.g., from a Lambda function) without CLI dependencies.

This architecture demonstrates several production engineering principles:
- **Separation of concerns**. each module has one job
- **Testability**. components can be tested in isolation
- **Reusability**. classes can be imported into other projects
- **Type safety**. Pydantic prevents an entire class of runtime errors
- **Configuration management**. settings centralized and easily changeable

### The Testing Strategy

With modular architecture came the responsibility of comprehensive testing. I built a PyTest suite with 5 test scenarios covering the full decision tree:

**Test 1. Normal Period:** Date with 0.8% conversion, similar to baseline. Expected: Status = OK, is_anomaly = False, consecutive.is_critical = False. Tests that the system doesnt throw false positives on healthy days.

**Test 2. Single Anomaly:** Date with 0.4% conversion (50% drop). Expected: Status = ANOMALY, alert_level = MEDIUM, is_anomaly = True, consecutive.is_critical = False (only 1 day). Tests that statistical threshold detection works but doesnt escalate prematurely.

**Test 3. Critical Period:** Third consecutive day with 0.1% conversion. Expected: Status = CRITICAL, alert_level = HIGH, consecutive.is_critical = True, streak_length = 3. Tests the consecutive day rule and alert escalation.

**Test 4. Insufficient Data:** Less than 4 weeks of historical data available. Expected: Status = INSUFFICIENT_DATA, alert_level = NONE. Tests graceful degradation when baseline cant be calculated reliably.

**Test 5. No Events:** Target date exists in timeline but has no events. Expected: Status = NO_DATA, alert_level = NONE. Tests handling of missing data scenarios.

### Why Synthetic Test Data?

A critical decision was using synthetic test data instead of the real 2.7M event Parquet files. This deserves explanation because it goes against the intuition of "test with real data."

**The problem with production data in tests:**

*Portability*. The Parquet files are 150MB and not in the Git repository. Tests would fail on CI/CD systems (GitHub Actions) where those files dont exist. Other developers cloning the repo wouldnt have the data.

*Speed*. Loading 2.7M events takes ~10 seconds. With 5 tests, thats 50 seconds just for data loading. Run tests 20 times while developing (normal during debugging), and youve wasted 16 minutes waiting. Synthetic data creates 5,000 events in 0.1 seconds. 5 tests complete in under 5 seconds total.

*Control*. Production data is messy. Which date has exactly 0.8% conversion? Which has exactly 3 consecutive anomalies? Youd have to search through the data or just hope suitable examples exist. Synthetic data lets you create exactly the scenario you want to test: "generate a day with 5,000 views and 40 transactions, giving exactly 0.8% conversion."

*Isolation*. If someone modifies the production Parquet files (adding more events, fixing data quality issues), tests break even though the code is correct. Synthetic data is generated fresh each test run and cant be accidentally modified.

*Edge cases*. How do you test "zero events for a date" with production data? Delete events? Create a gap in the timeline? With synthetic data, you just return an empty DataFrame.

The test suite uses fixtures. reusable functions that create data on-demand:

**historical_baseline_events**. 45 days of normal operation, ~0.83% conversion, realistic view/transaction counts. Provides the baseline for all tests.

**normal_events**. Single day with 0.8% conversion (5,000 views, 40 transactions). Tests normal operation detection.

**single_anomaly_events**. Single day with 0.4% conversion (4,500 views, 18 transactions). Tests anomaly detection without escalation.

**critical_period_day1_and_2_events**. Two days with 0.2% conversion (4,000 views, 8 transactions each). Sets up for consecutive detection.

**critical_period_day3_events**. Third day with 0.1% conversion (4,000 views, 4 transactions). Completes 3-day streak for critical alert.

Each fixture creates realistic event data matching the production schema: timestamp, event type, browser, OS, device, visitor ID, item ID, transaction ID. The data looks real, its just smaller and precisely controlled.

### Test Results

All 5 tests passed in 4.39 seconds. Heres what that proves:

✅ Normal days correctly identified (no false positives)  
✅ Single anomalies detected and correctly classified as MEDIUM alert  
✅ Critical periods (3+ consecutive days) trigger HIGH alerts  
✅ Edge cases handled gracefully (insufficient data, no data)  
✅ Statistical thresholds working correctly (±2σ with MAD)  

The test suite gives confidence that the system works as designed. More importantly, it demonstrates to potential employers that I understand software engineering beyond just "write code that runs once." Production systems need tests. Modular architecture needs tests. This project has both.

### Deliberate Non-Implementation: Logging and Error Handling

You might notice the production code lacks extensive try/except blocks, logger initialization, and custom exception classes. This is intentional, not an oversight.

What the project demonstrates:
- Clean test architecture with realistic scenarios
- Business logic (conversion rates, baselines, statistical thresholds)
- Deterministic synthetic data generation
- Clear status semantics (OK/ANOMALY/CRITICAL/NO_DATA)
- Pydantic models for type safety
- Modular class design

Adding comprehensive logging and error handling would shift focus from "understands business analytics and statistical methods" to "knows Python boilerplate." For a data engineering portfolio, the business logic and architecture are what matter. When asked in interviews "how would you productionize this?" I can discuss adding structured logging (with correlation IDs for request tracing), error handling strategies (retry logic, dead letter queues), monitoring (CloudWatch metrics, DataDog), and alerting (PagerDuty integration). But implementing all that in a portfolio project obscures the core statistical and architectural skills Im demonstrating.

The choice is documented in the README as an explicit architectural decision, not an accidental omission.

---

## Technical Skills Demonstrated

### Data Engineering
- **Apache Kafka:** Topic management, producer/consumer patterns, offset handling, consumer groups
- **Docker & Docker Compose:** Multi-container orchestration, networking (internal vs external listeners), volume mounts, environment variables
- **PostgreSQL/TimescaleDB:** Hypertable creation, authentication configuration (pg_hba.conf), connection pooling
- **Parquet Format:** Columnar storage, compression (Snappy), schema evolution, integration with analytics tools

### Statistical Analysis & Data Science
- **Robust Statistics:** Median vs mean for contaminated datasets, MAD (Median Absolute Deviation) for outlier resistance
- **Anomaly Detection:** Statistical thresholds (±2σ), same-weekday baseline to handle cyclical patterns, consecutive day rules to reduce false positives
- **Conversion Rate Analysis:** Funnel metrics, normalization for traffic variance, multi-dimensional slicing (device × OS × browser)
- **Hypothesis Testing:** Systematic elimination of alternative explanations (traffic drop, device changes, seasonal effects)

### Software Engineering
- **Object-Oriented Design:** Separation of concerns, single responsibility principle, dependency injection through composition
- **Type Safety:** Pydantic models for runtime validation, clear contracts between components, self-documenting code
- **Testing:** PyTest fixtures for reusable test data, synthetic data strategy for speed and control, 5 test scenarios covering decision tree
- **Modular Architecture:** Config management, data access layer separation, processor classes for business logic, pipeline orchestration

### Tools & Technologies
- **Python Libraries:** Pandas (vectorized operations, data transformations), NumPy (array operations, random sampling), kafka-python (streaming), psycopg2 (database connectivity), PyArrow (Parquet I/O), Pydantic (validation)
- **Power BI:** DAX measures for statistical calculations, data modeling (relationships, calculated columns), visualization best practices
- **Git & Version Control:** Commit discipline, .gitignore for large files, README documentation
- **Command Line Tools:** Kafka console producer/consumer, Docker exec for container debugging, PostgreSQL psql client

### Business & Analytics Acumen
- **E-commerce Metrics:** Conversion funnels (view → cart → transaction), baseline establishment, seasonality handling (day-of-week patterns)
- **Stakeholder Communication:** Statistical concepts explained without jargon (±2σ becomes "95% confidence"), actionable insights (specific browser/OS/device combination), timeline for detection improvement (3 days vs 2 weeks)
- **Root Cause Analysis:** Systematic dimensional drill-down, elimination of alternative hypotheses, validation through multi-combination testing
- **Production Thinking:** Pragmatic pivots (Parquet over PostgreSQL after 7 hours), balancing perfectionism with delivery, explicit documentation of architectural decisions

---

## Key Learnings & Insights

**Pragmatism beats perfectionism in real projects.** Seven hours debugging ODBC drivers taught me when to pivot. The project goal wasnt "master PostgreSQL ODBC". it was "build anomaly detection for e-commerce." Parquet achieved the same outcome in 30 minutes. This of course has to take into account the tools the cusomer offers and that should be used, for example for regulatory reasons.

**Robust statistics matter when data is contaminated.** The baseline contamination problem was subtle. mean-based calculations failed because the lookback window included bug days. Switching to median + MAD wasnt just "better". it was the difference between a system that works and one that doesnt.

**Architecture communicates professionalism.* The refactor from 500-line script to modular OOP design took extra time but transformed the project from "I can write code" to "I can design systems." Clean separation of concerns, type safety with Pyndantic and comprehensive testing prove production readiness.

**Streaming architecture without streaming requirements.** The Kafka pipeline was arguably nor necessary for a batch analytics problem. But it purpusefully demonstrated understanding of event-driven architecture, producer/consumer patterns, and offset management, skills that transfer directly to production systems at scale. Sometimes the journey is more valuable than the destination.

**Transparency builds trust.** Being upfront about synthetic bug injection, explaining the pivot from Microsoft Fabric to Kafka, documenting the Parquet decision. this honesty strengthens rather than weakens the portfolio. It shows I understand the difference between demo projects and production systems, and Im not trying to hide limitations.

---

## Project Outcomes

✅ **Complete streaming data pipeline** with Docker Compose (Kafka, Zookeeper, Consumer)  
✅ **Automated anomaly detection system** using robust statistics (median + MAD baseline, ±2σ threshold)  
✅ **Production-ready architecture** with OOP design, Pydantic models, and comprehensive tests  
✅ **Multi-combination validation** proving Safari iOS as isolated platform-specific bug  
✅ **79% faster detection** (3 days vs 2 weeks) through automated statistical monitoring  

**Root Cause Identified:** Safari iOS users experiencing 43% conversion rate drop (0.74% → 0.42%) due to checkout failure, confirmed through systematic dimensional analysis and multi-combination validation.

**Technical Demonstration:** End-to-end data engineering (streaming → storage → analytics → alerting), statistical rigor (robust statistics, hypothesis testing), software engineering (modular design, type safety, testing), and business impact (quantified detection improvement, actionable insights).

---

**Status:**  Project Complete. Ready for takeoff!!
