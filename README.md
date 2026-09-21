AIOps Assessment: Payment Service Monitoring

This repository is a small AIOps workflow for monitoring a payment-service. It
processes service telemetry and application logs, detects abnormal behavior, and
publishes anomaly events for downstream consumption.

Scenario

The monitored service handles payment requests. Most sample records show normal
operation: response times are about 120-150 ms, CPU and memory utilization are
moderate, and the service reports successful requests.

The operational problem is a short degradation at 10:05-10:06. Payment request
latency rises above 600 ms, CPU reaches 94%, memory reaches 91%, and the logs
report payment and database connection timeouts. These symptoms can indicate a
performance or dependency issue that operators need to identify quickly.

Purpose of AIOps

In this assessment, AIOps automatically evaluates operational telemetry and logs,
flags records that exceed configured thresholds, and turns those findings into
structured anomaly events. This reduces the need to inspect every record
manually and provides a machine-readable signal for downstream automation or
incident response.

Component Map

Operational data: data/service_data.json contains timestamped records for the
payment service, including response time, CPU, memory, log level, and log message.

Metrics and logs: Each record in data/service_data.json is the combined metrics
and log input used by the workflow.

Anomaly detection: src/anomaly_detector.py applies response-time, CPU, and memory
thresholds and includes relevant log conditions in each anomaly's reasons.

Event production: src/event_producer.py publishes detected anomaly dictionaries
to an EventTopic.

Event topics: src/event_topic.py provides the in-memory topic abstraction that
stores, returns, and clears messages.

Event consumption: src/event_consumer.py reads messages from a topic for downstream
processing.

Final AIOps processing: src/aiops_pipeline.py loads the operational data, runs
detection, produces events, consumes events, and reports records processed and
anomalies found.

src/calculations.py and its tests are supporting assessment examples and are
not part of the payment-service AIOps flow.

Operational data observations

The file data/service_data.json contains ten observations for payment-service.
Each observation is made up of three metric fields: response_time_ms records how
long the request took, cpu_percent records CPU use, and memory_percent records
memory use. The service field identifies the application being monitored.

The log information is in log_level and message. The normal records have an INFO
level and say that the payment request was processed successfully. The two
problem records have an ERROR level. One reports a payment service timeout and
the other reports a database connection timeout.

The timestamp field records when each observation was taken. The timestamps use
an ISO-style date and time and are spaced one minute apart, from 10:00 through
10:09 on 20 September 2026. This makes it possible to see the short-lived change
in behaviour and what happened immediately before and after it.

The observations from 10:00 to 10:04 look normal. Response times stay between
120 and 142 milliseconds, CPU stays between 42 and 48 percent, and memory stays
between 51 and 55 percent. The observations from 10:07 to 10:09 also look normal
and show that the service returned to its earlier range after the incident.

The observations at 10:05 and 10:06 look unusual. Response time jumps to 610
and 640 milliseconds. At 10:06, CPU reaches 94 percent and memory reaches 91
percent; both are well above the normal pattern. The error messages at those
same times provide useful context, pointing first to a payment timeout and then
to a database connection timeout. The supplied detector uses thresholds for the
three numeric metrics, so it identifies the high response times and the high
CPU and memory values. It does not currently use the ERROR log level as a
detection reason, even though the log entries are clearly relevant to the
operational diagnosis.

## Detection report

The unchanged pipeline processed all ten observations and internally detected
two anomalies. The first was at 2026-09-20T10:05:00. Its response time was 610
milliseconds, which crossed the 500 millisecond threshold. CPU was 75 percent
and memory was 70 percent, so neither resource metric crossed its threshold.
The record contained an ERROR log with the message Payment service timeout, but
the detector did not include that log as a reason because it checks for WARNING
rather than ERROR.

The second anomaly was at 2026-09-20T10:06:00. Its response time was 640
milliseconds, CPU was 94 percent, and memory was 91 percent. These values
crossed all three configured thresholds. The record also contained an ERROR log
with the message Database connection timeout, but this log event was missed by
the detector for the same reason.

The records from 10:00 to 10:04 and 10:07 to 10:09 were treated as normal and
were not flagged. No normal event was incorrectly flagged in this data set. Both
expected anomalous observations were detected through their metrics, but both
relevant ERROR log events were missed. The final pipeline output reported zero
events consumed because the producer uses the service-events topic while the
consumer listens to the separate anomaly-events topic, so the printed report did
not display the detected event details.

One limitation is that the detector relies on fixed thresholds and does not
learn what normal behaviour looks like for this service. A possible improvement
would be to build a baseline from recent healthy observations and combine that
with explicit checks for ERROR logs and timeout messages.

## Event flow verification

The provided workflow was executed without changing the event components. It
processed 10 records and identified 2 anomaly events, at 10:05 and 10:06.

The detector creates an event when a record crosses a configured metric
threshold. The event contains the timestamp, service, type, reasons, and source
record. The producer receives each event and returns True after publishing it.
The topic stores the published messages in memory. In the execution, both
producer.publish calls returned True and the service-events topic contained 2
messages.

The consumer is intended to read those messages and pass them to the downstream
AIOps pipeline. In the supplied workflow, however, the consumer was connected
to the separate anomaly-events topic. It therefore received 0 messages, and the
printed workflow result reported Events consumed: 0. The events did not reach
the downstream processing step, so the complete event flow could not be
verified as successful with the code as provided.

The component roles are: the event is the anomaly message created by detection;
the producer publishes that message; the topic stores and makes messages
available; the consumer reads messages from its topic; and the downstream AIOps
component is the pipeline that reports consumed events. The producer-to-topic
stage was verified, but the topic-to-consumer and downstream stages were not
completed because the two sides use different topic names.
