# AIOps Operational Data Analysis and Workflow Validation

## Scenario
This repository models a lightweight AIOps workflow for a payment service. Synthetic operational data is used to represent a service’s normal and abnormal runtime behaviour. The system detects anomalies from runtime metrics and error logs, publishes them as events, and validates that the event can move through a minimal producer/topic/consumer pipeline before the downstream AIOps component consumes the result.

## Operational data description
The repository includes a synthetic telemetry dataset in [data/service_data.json](data/service_data.json). Each observation contains:
- timestamp: ISO-8601 timestamp for the sample time
- service: affected service name
- response_time_ms: service latency in milliseconds
- cpu_percent: CPU usage percentage
- memory_percent: memory usage percentage
- log_level: log severity (INFO, ERROR, etc.)
- message: log message text

## Observations from logs and metrics
### Metrics
The metric fields are:
- response_time_ms
- cpu_percent
- memory_percent

These are numeric operational indicators that show service health and resource pressure.

### Log information
The log-related fields are:
- log_level
- message

These fields describe what the service reported at the same point in time.

### Timestamp usage
The timestamps in the dataset are sequence-based and roughly one minute apart, starting at 2026-09-20T10:00:00 and ending at 2026-09-20T10:09:00. They are used to order observations chronologically and correlate metric spikes with log events over time.

### Normal behaviour
The normal observations are the initial records and the recovery records after the anomaly window. Examples:
- 2026-09-20T10:00:00 to 2026-09-20T10:04:00: response times stay around 120-145 ms, CPU around 42-50%, memory around 51-57%, and logs are INFO with successful requests.
- 2026-09-20T10:07:00 to 2026-09-20T10:09:00: values return to a stable range and messages indicate successful processing.

These records show stable latency, moderate resource usage, and successful service behaviour.

### Unusual behaviour
The unusual observations are the records around 10:05 and 10:06:
- 2026-09-20T10:05:00: response_time_ms = 610, cpu_percent = 75, memory_percent = 70, log_level = ERROR, message = "Payment service timeout"
- 2026-09-20T10:06:00: response_time_ms = 640, cpu_percent = 94, memory_percent = 91, log_level = ERROR, message = "Database connection timeout"

These are anomalous because they combine sustained latency spikes, high CPU and memory usage, and ERROR-level messages indicating service failure conditions.

## Anomaly-detection findings
The detection logic in [src/anomaly_detector.py](src/anomaly_detector.py) flags a record when any of the following occurs:
- response_time_ms exceeds 500
- cpu_percent exceeds 80
- memory_percent exceeds 80
- log_level is ERROR

With the provided dataset, the detector identifies two abnormal records:
- 2026-09-20T10:05:00, payment-service
- 2026-09-20T10:06:00, payment-service

Relevant reasons include:
- High response time
- High CPU utilization
- High memory utilization
- Error log detected

The detector distinguishes normal records from anomalous ones by requiring at least one anomaly trigger. This successfully separates the healthy early and late samples from the failing mid-window samples.

### Missed or false positives
- Expected anomaly coverage: the detection identifies both clearly anomalous points and does not flag the obvious healthy intervals.
- Normal event incorrectly flagged: none in the provided sample set.
- Potential limitation: the detector is rule-based and threshold-driven; it will miss more subtle degradations that do not cross those thresholds or produce error-level logs.

## Event-processing flow
The workflow is implemented in [src/aiops_pipeline.py](src/aiops_pipeline.py) with the following components:
- EventTopic: in-memory topic storing messages
- EventProducer: sends anomaly events to the configured topic
- EventConsumer: reads messages from the topic
- Event/message: produced anomaly payload containing timestamp, service, type, reasons, and source record

The event flow is:
Operational data -> Anomaly detection -> Event -> Producer -> Topic -> Consumer -> downstream AIOps component

## Final workflow execution result
I verified the corrected workflow by running the project’s test suite and the pipeline execution.

Command used:
- python -m pytest -q
- python -m src.aiops_pipeline

Result:
- 8 tests passed
- The pipeline processed all 10 records
- 2 anomalies were detected
- 2 anomaly events were published to the in-memory topic
- 2 messages were consumed successfully
- The final output represented the payment-service outage conditions with timestamps and reasons for the anomaly.

## Issues identified and corrected
I fixed two workflow issues:
1. Incorrect topic wiring in [src/aiops_pipeline.py](src/aiops_pipeline.py): the producer and consumer were connected to different topics, which prevented the event from being consumed after it was published.
2. Incorrect anomaly rule in [src/anomaly_detector.py](src/anomaly_detector.py): the detector was mistakenly flagging WARNING logs as errors instead of ERROR logs.
3. Python module import compatibility: the source modules were updated to support both package execution and direct script execution so the project works correctly under the standard Python import model.

## Possible improvement
A practical improvement would be to add a baseline model for normal latency and resource use (for example, a rolling mean and standard deviation) so the system can detect anomalies that are statistically unusual even when they remain under the current fixed thresholds.

## Reproduction steps
1. Open a terminal in the repository root.
2. Install dependencies if needed:
   python -m pip install -r requirements.txt
3. Run the validation suite:
   python -m pytest -q
4. Run the workflow:
   python -m src.aiops_pipeline
5. Review the printed anomaly summary and verify that the anomaly events are detected and consumed.

---
&copy; 2025 GitHub &bull; [Code of Conduct](https://www.contributor-covenant.org/version/2/1/code_of_conduct/code_of_conduct.md) &bull; [MIT License](https://gh.io/mit)

