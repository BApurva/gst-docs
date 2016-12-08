# Live sources

A live source is a source that cannot be arbitrarily `PAUSED` without
losing data.

A live source such as an element capturing audio or video need to be
handled in a special way. It does not make sense to start the dataflow
in the `PAUSED` state for those devices as the user might wait a long time
between going from `PAUSED` to PLAYING, making the previously captured
buffers irrelevant.

A live source therefore only produces buffers in the PLAYING state. This
has implications for sinks waiting for a buffer to complete the preroll
state since such a buffer might never arrive.

Live sources return `NO_PREROLL` when going to the `PAUSED` state to inform
the bin/pipeline that this element will not be able to produce data in
the `PAUSED` state. `NO_PREROLL` should be returned for both READY→PAUSED
and PLAYING→PAUSED.

When performing a get\_state() on a bin with a non-zero timeout value,
the bin must be sure that there are no live sources in the pipeline
because else the get\_state() function would block on the sinks.

A gstbin therefore always performs a zero timeout get\_state() on its
elements to discover the `NO_PREROLL` (and ERROR) elements before
performing a blocking wait.

## Scheduling

Live sources will not produce data in the paused state. They block in
the getrange function or in the loop function until they go to PLAYING.

## Latency

The live source timestamps its data with the time of the clock at the
time the data was captured. Normally it will take some time to capture
the first sample of data and the last sample. This means that when the
buffer arrives at the sink, it will already be late and will be dropped.

The latency is the time it takes to construct one buffer of data. This
latency is exposed with a `LATENCY` query.

See [latency](design/latency.md)

## Timestamps

Live sources always timestamp their buffers with the running\_time of
the pipeline. This is needed to be able to match the timestamps of
different live sources in order to synchronize them.

This is in contrast to non-live sources, which timestamp their buffers
starting from running\_time 0.