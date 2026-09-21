AIOps Assessment: Payment Service Monitoring

Scenario

This assessment monitors a payment-service that processes payment requests. The
operational problem is a short performance degradation. At 10:05 and 10:06,
payment requests become slow and the service reports timeout errors. AIOps is
used to examine the service data, identify unusual behaviour, create anomaly
events, and pass those events through the event-processing workflow.

Operational data

The operational data is in data/service_data.json. It contains ten records for
payment-service, with one record per minute from 10:00 to 10:09 on 20 September
2026.

The metric fields are response_time_ms, cpu_percent, and memory_percent. The
log fields are log_level and message. The timestamp field shows when each
observation was recorded and makes the change in behaviour visible over time.

Observations from the data

The records from 10:00 to 10:04 and 10:07 to 10:09 appear normal. Response time
is between 120 and 150 milliseconds, CPU is between 42 and 50 percent, and
memory is between 51 and 57 percent. Each of these records has an INFO level
and the message Payment request processed successfully.

The records at 10:05 and 10:06 appear unusual. Response time rises to 610 and
640 milliseconds. At 10:06, CPU reaches 94 percent and memory reaches 91
percent. The logs are also concerning: 10:05 reports Payment service timeout and
10:06 reports Database connection timeout, both at ERROR level.

Anomaly detection

src/anomaly_detector.py uses these thresholds:

Response time above 500 milliseconds is anomalous.
CPU above 80 percent is anomalous.
Memory above 80 percent is anomalous.
WARNING or ERROR log levels are reported as concerning log events.

The corrected detector identifies two anomalies. The event at 10:05 has a 610
millisecond response time and an ERROR log, so its reasons are High response
time and Error log detected. The event at 10:06 has a 640 millisecond response
time, 94 percent CPU, 91 percent memory, and an ERROR log, so its reasons are
High response time, High CPU utilization, High memory utilization, and Error
log detected.

The normal observations were not flagged. No expected anomaly was missed after
the correction, and no normal event was incorrectly flagged in this data set.

Event-processing flow

The detector creates an anomaly event containing the timestamp, service, type,
reasons, and original source record. EventProducer receives the event and
publishes it to EventTopic. EventTopic stores the messages in memory. The
EventConsumer reads messages from that same topic. The AIOps pipeline then
reports the consumed events as the final output.

The complete flow is:

Operational data -> anomaly detection -> anomaly event -> producer -> topic ->
consumer -> final AIOps output

Final workflow result

The corrected workflow was run with:

PYTHONPATH=src python3 src/aiops_pipeline.py

It produced this result:

Records processed: 10
Anomalies detected: 2
Events consumed: 2

The final output displayed both payment-service anomaly events. This confirms
that the operational data was processed, abnormal behaviour was detected,
events were generated and published, the consumer received them, and the final
AIOps component reported the operational issue.

Issues identified and corrected

The first issue was in src/anomaly_detector.py. The data contained ERROR logs,
but the original detector checked only for WARNING. Therefore the metric
anomalies were detected, but the relevant error-log information was missed. The
correction was to recognize ERROR as well as WARNING.

The second issue was in src/aiops_pipeline.py. The producer published to a
service-events topic, while the original consumer listened to a separate
anomaly-events topic. The original run processed 10 records, detected 2
anomalies, but consumed 0 events. The correction was to construct the consumer
with the same service-events topic used by the producer.

The original incorrect output was:

Records processed: 10
Anomalies detected: 2
Events consumed: 0

After the corrections, the workflow reported two consumed events and displayed
both anomaly details. The corrections use the existing detector, producer,
topic, consumer, and pipeline architecture; no replacement implementation was
introduced.

Limitation and possible improvement

The detector uses fixed thresholds, so it does not learn the normal behaviour
of this service or adapt to changes in traffic. A possible improvement would be
to establish a baseline from recent healthy observations and combine it with
explicit ERROR-log and timeout checks.

Reproducing the demonstration

Run these commands from the repository root. Capture the terminal output after
each command if screenshots are required for submission.

1. View the operational data, metrics, and logs:

python3 -m json.tool data/service_data.json

2. Run the complete AIOps workflow:

PYTHONPATH=src python3 src/aiops_pipeline.py

3. Run the validation tests:

env PYTHONPATH=$PWD:$PWD/src python3 -m pytest

The expected test result is 8 passed. The workflow result should show 10
records processed, 2 anomalies detected, and 2 events consumed.
