# Clickstream_Project Readme and Project overview
> **An end-to-end data engineering project tackling a real business problem: identifying the root cause of a 35% conversion rate drop for a fictional electronics retailer**

---
## Purpose & Context

This multi-stage project around the business problem of a fictional electronic device seller covers a wide range of duties in the data engineering and data science domain.
Oftentimes the solutions we provide as ML Engineers or Data Scientists in portfolio projects are somewhat dispatched from the real world in the sense that they cover isolated problems, sometimes with implementation. In the real world this is rarely the case though, at least if you want to work end-to-end. While end-to-end might sound buzzworthy, it really means anything from the business problem & definition of goals until the actual usage of the tool by the users after deployment.
This project ties together different tech-sulitions with a congruent business case to show that it is possible to create a whole project even without a team.

## Project Stages Overview

The project consists of three connected stages, each with it's own repository and README. If you are interested in one or more of the three stages, feel free to visit the corresponding repo to see the full code as well as a comprehensive readme.

> Stage 1: Foundation: Data Preparation, Statistical Analysis & Root Cause Identification
> 
> Stage 2: Streaming Infrastructure: Event Streaming Producer and Comsumer
> 
> Stage 3: Production: Modular Detection System & Comprehensive Testing


## Business Scenario

The business scenario is intentionally realistic: an e-commerce company notices conversion rates tanking in their dashboards but has no idea why. Marketing blames the product team, product blames engineering which points at third-party scripts. Everyones looking at different metrics and talking past each other. That's the chaos data engineering exists to solve.

The client reaches out to me for (initially!) two different tasks:
1. Highest prio: Find the reason why the conversion drops latel
2. Install a solution to read clickstreams and convert them into an analyzable format, as right now this is done only with samples in excel

## **Stage 1: Foundation. Data Preparation, Streaming Infrastructure, Statistical Analysis**
**Repository:** [`Clickstream_project_part_1_exploration_and_data_preperation`](https://github.com/ksokoll/Clickstream_project_part_1_exploration_and_data_preperation)  

Before we can start with the actual use-case, we need to prepare some groundwork. I used the Retail Rocket dataset (2,7M e-commerce events collected over 4,5 months), but added a bit of secred sauce: I contaminated the dataset with a realistic bug: Users with a specific combination of OS, device and browser could not proceed to checkout, resulting in slightly, but measurebly rising number of abadoned carts.

The contamination was more difficult then I expected, but eventually ended up with a good result.

Not it's time to tacke the first of the two tasks given by my client: Finding the root cause for the Conversion drop.

After getting data into Parquet files (pivoting from TimescaleDB after ODBC encoding hell), I created a Power BI dashboard and conducted systematic root cause analysis. This wasn't just "make some charts and see what looks weird", but more hypothesis-driven investigation using statistical rigor. Starting broad, total traffic dropped from 14,390 events/day to 12,450 in the anomaly period. Sounds significant until you calculate standard deviation (4,440 events) - the drop is well within ±1σ, meaning normal variance. Traffic wasn't the problem.

Then the device was sliced: Desktop and Tablet normal, but Mobile transactions down 33% and sitting below -1 sigma. Narrowing to operating systems: Android, Windows, macOS all fine. iOS? Crashed from 33.5 transactions/day to 10.25. That's -4 sigma, a statistical blazing red flag.

The analysis of the browsers confirmed Safari was borderline (normal range but at the lower bound). The smoking gun was the combination: Mobile x iOS x Safari showed views *up* 18%, addtocart *up* 37%, but transactions *down* 33%. Users were browsing more and trying to buy more, but checkout was failing.

Conversion rate quantified the damage: 0.74% normal → 0.42% during bug = 43 percentage point drop. For e-commerce, this is catastrophic.

I then built an automated anomaly detection script using same-weekday baselines (compare each Saturday to the last 4 Saturdays, accounting for day-of-week patterns) and robust statistics. Initial implementation with mean/standard deviation failed because the lookback window was contaminated with bug days. Switching to median + MAD (Median Absolute Deviation) solved it - these metrics ignore outliers and gave reliable baselines even when recent data was corrupted.

## **Stage 2: Kafka Clickstream Producer & Consumer**
**Repository:** [`Clickstream_project_part_2_kafka_stream`](https://github.com/ksokoll/Clickstream_project_part_2_kafka_stream)

Next, we tackle step two of the project, which is the second task given by my client: Implementing a tool that is able to convert the streaming data of their provider into analyzable batched files.

Since this is a fictional client, there is no real event stream I could connect to. Therefore I built a event stream producer myself based on the above Retail Rocket dataset. Setting it up was quite easy, the more difficult part was the following:

For the consumer, over the course of the process I pivoted back from using MS Fabric event streams, as planned originally, to a local Kafka stack after locking myself out of Azure with a MFA lock. Thats something that can happen when you try to use enterprise software as a single private developer, but Kafka was doing the job also quite well. In the end switching to Kafka resulted in cool lessons about producer/consumer patterns, batching, offset management, containerization, etc.

It took some serious time to handle all the errors, but in the end it worked out and it was nice to see both producer and cosumer in action!

## **Stage 3: Production - Modular Detection System & Comprehensive Testing**
**Repository:** [`Clickstream_project_part_3_anomaly_detection`](https://github.com/ksokoll/Clickstream_project_part_3_anomaly_detection)

In this stage I assumed a change of the clients planned tasks for me. Instead of just uncovering the anomaly and setting up a clickstream-converter, I got the task to build a tool that helps them identify such anomanlies in the future in a faster fashion, and for every OS/Browser/Device combination that they wished. This imitates typical client behavior with changing requirements as new results come to the surface during the project.

Therefore in stage 3 the script in stage two got refactured into a more clean, modular and testable production ready architecture, which looks like this:
- **DataLoader** handles Parquet file access (load all, load before date, load specific date)
- **ConversionCalculator** computes daily conversion rates for browser/OS/device combinations
- **BaselineCalculator** implements same-weekday median + MAD baseline logic
- **AnomalyDetector** applies ±2σ thresholds and detects consecutive-day streaks
- **Pipeline** orchestrates the workflow (load historical → calculate baseline → load new events → detect anomalies → check streaks → return structured result)
- **Main** provides CLI interface (separated from pipeline so it can be imported programmatically)

Pydantic was used for type safety, but I purposefully avoided any other kind of error-catching and logging to keep the focus of this project on the business logic, rather than too much software engineering.
A step that took some thought was testing: Two weeks of anomaly data are fine, but normally tests have to be portable, fast and controllabe. Therefore I created a small synthetic dataset to test all modes of the tool:

1. Normal period (0.8% conversion) → Status: OK
2. Single anomaly (0.4% conversion) → Status: ANOMALY, Alert: MEDIUM
3. Critical period (0.1% conversion, 3 consecutive days) → Status: CRITICAL, Alert: HIGH
4. Insufficient historical data (<4 weeks) → Status: INSUFFICIENT_DATA
5. No events for target date → Status: NO_DATA

All tests use pytest fixtures that generate realistic event data on-the-fly - thousands of view/addtocart/transaction events with proper timestamps, visitor IDs, and browser/OS/device combinations.

## Project Outcomes

**Business Impact:**
- Root cause identified: Safari iOS checkout bug causing 43% conversion rate drop
- Detection speed: 3 days (automated) vs 2 weeks (manual) = 79% improvement
- Event handler/consumer implemented that allows for more extensive reporting

**Technical Deliverables:**
- Streaming data pipeline with Kafka, Docker Compose, and Parquet persistence
- Power BI dashboard with statistical analysis and dimensional breakdowns
- Production-ready anomaly detection system with OOP architecture and comprehensive tests
- Complete documentation (README, project log, test results, statistical methodology)

**Skills Demonstrated:**
- End-to-end data engineering (ingestion → storage → analytics → alerting)
- Statistical rigor (robust statistics, hypothesis testing, multi-dimensional analysis)
- Software engineering (modular design, type safety, test-driven development)
- Business thinking (conversion funnels, root cause analysis)
- Statistical confidence: Used sigma standard deviations instead of gut feeling to identify anomaly periods

---

**This project demonstrates that complex data problems require more than just technical skills. They need systematic thinking, pragmatic decision-making, and the ability to communicate findings clearly. Each stage builds toward a complete solution that could actually be deployed and used.**

