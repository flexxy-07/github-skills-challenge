# AIOps Assessment: Payment Service Monitoring

This repository is a small AIOps workflow for monitoring a `payment-service`. It
processes service telemetry and application logs, detects abnormal behavior, and
publishes anomaly events for downstream consumption.

## Scenario

The monitored service handles payment requests. Most sample records show normal
operation: response times are about 120-150 ms, CPU and memory utilization are
moderate, and the service reports successful requests.

The operational problem is a short degradation at 10:05-10:06. Payment request
latency rises above 600 ms, CPU reaches 94%, memory reaches 91%, and the logs
report payment and database connection timeouts. These symptoms can indicate a
performance or dependency issue that operators need to identify quickly.

## Purpose of AIOps

In this assessment, AIOps automatically evaluates operational telemetry and logs,
flags records that exceed configured thresholds, and turns those findings into
structured anomaly events. This reduces the need to inspect every record
manually and provides a machine-readable signal for downstream automation or
incident response.

## Component Map

- **Operational data:** `data/service_data.json` contains timestamped records for
	the payment service, including response time, CPU, memory, log level, and log
	message.
- **Metrics and logs:** Each record in `data/service_data.json` is the combined
	metrics and log input used by the workflow.
- **Anomaly detection:** `src/anomaly_detector.py` applies response-time, CPU,
	and memory thresholds and includes relevant log conditions in each anomaly's
	reasons.
- **Event production:** `src/event_producer.py` publishes detected anomaly
	dictionaries to an `EventTopic`.
- **Event topics:** `src/event_topic.py` provides the in-memory topic abstraction
	that stores, returns, and clears messages.
- **Event consumption:** `src/event_consumer.py` reads messages from a topic for
	downstream processing.
- **Final AIOps processing:** `src/aiops_pipeline.py` loads the operational
	data, runs detection, produces events, consumes events, and reports records
	processed and anomalies found.

`src/calculations.py` and its tests are supporting assessment examples and are
not part of the payment-service AIOps flow.


```
