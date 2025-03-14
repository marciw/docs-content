# Health Report Pipeline Status [health-report-pipeline-status]

The Pipeline indicator has a `status` probe that is capable of producing one of several diagnoses about the pipeline’s lifecycle, indicating whether the pipeline is currently running.

## $$$loading$$$Loading Pipeline [health-report-pipeline-status-diagnosis-loading]

A pipeline that is loading is not yet processing data, and is considered a temporarily-degraded pipeline state. Some plugins perform actions or pre-validation that can delay the starting of the pipeline, such as when a plugin pre-establishes a connection to an external service before allowing the pipeline to start. When these plugins take significant time to start up, the whole pipeline can remain in a loading state for an extended time.

If your pipeline does not come up in a reasonable amount of time, consider checking the Logstash logs to see if the plugin shows evidence of being caught in a retry loop.


## $$$finished$$$Finished Pipeline [health-report-pipeline-status-diagnosis-finished]

A logstash pipeline whose input plugins have all completed will be shut down once events have finished processing.

Many plugins can be configured to run indefinitely, either by listening for new inbound events or by polling for events on a schedule. A finished pipeline will not produce or process any more events until it is restarted, which will occur if the pipeline’s definition is changed and pipeline reloads are enabled. If you wish to keep your pipeline runing, consider configuring its input to run on a schedule or otherwise listen for new events.


## $$$terminated$$$Terminated Pipeline [health-report-pipeline-status-diagnosis-terminated]

When a Logstash pipeline’s filter or output plugins crash, the entire pipeline is terminated and intervention is required.

A terminated pipeline will not produce or process any more events until it is restarted, which will occur if the pipeline’s definition is changed and pipeline reloads are enabled. Check the logs to determine the cause of the crash, and report the issue to the plugin maintainers.


## $$$unknown$$$Unknown Pipeline [health-report-pipeline-status-diagnosis-unknown]

When a Logstash pipeline either cannot be created or has recently been deleted the health report doesn’t know enough to produce a meaningful status.

Check the logs to determine if the pipeline crashed during creation, and report the issue to the plugin maintainers.


