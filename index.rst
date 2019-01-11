..
  Technote content.

  See https://developer.lsst.io/restructuredtext/style.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

DM-EFD prototype implementation based on Kafka and InfluxDB.

.. note::

   **This technote is not yet published.**


Introduction
============

The adopted technologies
========================

The Confluent Kafka platform
----------------------------

The InfluxData stack
--------------------

Why a time series database?
^^^^^^^^^^^^^^^^^^^^^^^^^^^


Why InfluxData?
^^^^^^^^^^^^^^^
The InfluxData stack provides a complete solution for storing, visualizing and processing time series data:

* `InfluxDB <https://docs.influxdata.com/influxdb/v1.7/>`_ is an open-source time series database written in Go, it has an SQL-like query language (InfluxQL) and provides an HTTP API as well as API client libraries.
* `Chronograf <https://docs.influxdata.com/chronograf/v1.7/>`_  is an open-source web application for time series visualization written in Go and React.js
* `Kapacitor <https://docs.influxdata.com/kapacitor/v1.5/>`_ is an open-source framework written in Go for processing, monitoring, and alerting on time series data.


Deploying Confluent Kafka and InfluxData
========================================

The GKE sluster specs
---------------------

* Machine type: n1-standard-2 (2 vCPUs, 7.5 GB memory)
* Size: 3 nodes
* Total cores: 6 vCPUs
* Total memory:	22.50 GB

Monitoring
----------

The SAL mock experiment
=======================

Designing the experiment
------------------------

============ ================= ============ =============== ===================================
Producer ID  Topic type        # of topics  Frequency (Hz)  Expected throughput (# messages/s)
============ ================= ============ =============== ===================================
`0`_         SAL Commands      274          1               274
`1`_         SAL Log Events    541          10              5410
`2`_         SAL Telemetry     236          100             23600
============ ================= ============ =============== ===================================

.. _`0`: https://github.com/lsst-sqre/kafka-efd-demo/blob/tickets/DM-17052/k8s-apps/salmock-1node-commands-1hz.yaml

.. _`1`: https://github.com/lsst-sqre/kafka-efd-demo/blob/tickets/DM-17052/k8s-apps/salmock-1node-logevents-10hz.yaml

.. _`2`: https://github.com/lsst-sqre/kafka-efd-demo/blob/tickets/DM-17052/k8s-apps/salmock-1node-logevents-10hz.yaml

Total number of topics: 1051

Total expected throughput: 29284 messages/s

Experiment Duration: 16h




Producing SAL topics
--------------------

Converting SAL XML schema to Apache Avro
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

AIOKafkaProducer
^^^^^^^^^^^^^^^^

The measured throughput
^^^^^^^^^^^^^^^^^^^^^^^

.. figure:: /_static/salmock_produced_total.png
   :name: Producer metric.
   :target: _static/salmock_produced_total.png

   The producer throughput as measured by the ``salmock_produced_total`` metric.

Number of topics produced: 1051

Maximum measured throughput: 1330 messages/s



Connecting Kafka and InfluxDB
-----------------------------

At the time of this implementation, the `Confluent InfluxDB connector <https://docs.confluent.io/current/connect/kafka-connect-influxdb/index.html>`_ was still in preview and did not have the functionality we needed. Instead of the Confluent connector, we used the `InfluxDB Sink connector developed by Landoop <https://docs.lenses.io/connectors/sink/influx.html>`_.

We added the Landoop InfluxDB Sink connector 1.1 plugin to the ``cp-kafka-connect`` container, and implemented scripts to facilitate its configuration.

A limitation of version 1.1, though, was the lack of support for Avro ``array`` data type, which was solved by `contributing to the plugin development <https://github.com/Landoop/stream-reactor/pull/522>`_.



The InfluxDB data model
^^^^^^^^^^^^^^^^^^^^^^^

The InfluxDB data model does not necessarily need to follow the Avro schema. The InfluxDB Sink Connector supports KCQL, the Kafka Connect Query Language that can be used to select fields, define the target measurement, and `set tags to annotate the measurements <https://docs.influxdata.com/influxdb/v1.7/concepts/schema_and_data_layout/>`_.

In the current implementation, the InfluxDB data model is the simplest possible. We just create an InfluxDB measurement with the topic name and select all fields from the topic.

Example of Avro schema for the ``"MTM1M3_accelerometerData`` SAL telemetry topic, and the corresponding InfluxDB measurement:

::

  {
    "fields": [
      {
        "doc": "Timestamp when the Kafka message was created.",
        "name": "kafka_timestamp",
        "type": {
          "logicalType": "timestamp-millis",
          "type": "long"
        }
      },
      {
        "name": "timestamp",
        "type": "double"
      },
      {
        "name": "rawAccelerometers",
        "type": {
          "items": "float",
          "type": "array"
        }
      },
      {
        "name": "accelerometers",
        "type": {
          "items": "float",
          "type": "array"
        }
      },
      {
        "name": "angularAccelerationX",
        "type": "float"
      },
      {
        "name": "angularAccelerationY",
        "type": "float"
      },
      {
        "name": "angularAccelerationZ",
        "type": "float"
      }
    ],
    "name": "MTM1M3_accelerometerData",
    "namespace": "lsst.sal",
    "sal_subsystem": "MTM1M3",
    "sal_topic_type": "SALTelemetry",
    "sal_version": "3.8.35",
    "type": "record"
  }


::

    > SHOW FIELD KEYS FROM "mtm1m3-accelerometerdata"
    name: mtm1m3-accelerometerdata
    fieldKey             fieldType
    --------             ---------
    accelerometers0      float
    accelerometers1      float
    angularAccelerationX float
    angularAccelerationY float
    angularAccelerationZ float
    kafka_timestamp      integer
    rawAccelerometers0   float
    rawAccelerometers1   float
    timestamp            float

.. note::

  1. InfluxDB does not have ``double`` or ``long`` `datatypes <https://docs.influxdata.com/influxdb/v1.7/write_protocols/line_protocol_reference/#data-types>`_.
  2. InfluxDB does not suppot derived data types like ``arrays``. Fields named like ``<field name>0, <field name>1, ...`` were extracted from arrays in the Avro message.



In  `Chronograf <https://chronograf-demo.lsst.codes>`_, one can browse the SAL topics and visualize them using the Explore tool.


.. figure:: /_static/chronograf.png
   :name: Chronograf Explore tool.
   :target: _static/chronograf.png

   Visualization of a particular topic using the Chronograf Explore tool.

These visualizations can be organized in Dashboards for monitoring the commands, log events and telemetry data from the different subsystems.

Latency measurements
^^^^^^^^^^^^^^^^^^^^

.. figure:: /_static/latency.png
   :name: Roundtrip latency for a telemetry message.
   :target: _static/latency.png

   The roundtrip latency for a telemetry topic measured as the difference between the InfluxDB and Kafka timestamps.

We characterize the roundtrip latency as the difference between the time when the message was produced and the time when it was written to InfluxDB.

**The median roundtrip latency for a telemetry topic produced over the duration of the experiment was 183ms with 99% of the messages with latency smaller than 1.34s.**

This result would allow for quasi-realtime access to the telemetry stream from resources at the LDF.  This would not be possible with the current baseline design (see discussion in `DMTN-082 <https://dmtn-082.lsst.io/>`_).

In particular, it is very encouraging because both Kafka and InfluxDB were deployed in modest hardware, and with default configurations. There is certainly room for improvement, and many aspects to explore in both Kafa and InfluxDB deployments.

The InfluxDB throughput
^^^^^^^^^^^^^^^^^^^^^^^

.. figure:: /_static/influxdb.png
   :name: InfluxDB throughput.
   :target: _static/influxdb.png

   InfluxDB throughput measured as number of points per minute.

An InfluxDB database stores points. In the current data model a point has a timestamp, a measurement, and its fields. Thus, by construction an InfluxDB point is equivalent to a single message.

The measured InfluxDB throughput during the experiment was ~80k points per minute or ~1.3k messages/s, which is basically determined by the producer throughput (see above).

However, the measured producer throughput is lower than expected throughput. We could improve the performance of the producer code or put more resources to run the producer jobs, but a simple test can be done to assess the maximum InfluxDB throughput.

If we stop the InfluxDB Sink connector, and let the producer to run for a period of time T, the messages produced during T will be cached at the Kafka broker. As soon as the connector is res-started, all the messages will be flushed to InfluxDB as if they were produced in a much higher throughput. The result of this test is shown in the  figure below, were we see a measured throughput of 1M points/minute (or 16k messages/s) about 10x higher than the previous result.


.. figure:: /_static/influxdb_max.png
   :name: InfluxDB maximum throughput.
   :target: _static/influxdb_max.png

   InfluxDB maximum throughput measured as number of points per minute.

Again, this result is pretty good given that we are running on modest hardware and using the InfluxDB default configuration.

Configuring a Retention Policy
------------------------------



Downsampling time series
------------------------

.. figure:: /_static/downsampling.png
   :name: Downsampling a SAL Telemetry message using a continuous query.
   :target: _static/downsampling.png

Example of a continuous query:

::

  CREATE continuous query "mtm1m3-accelerometerdata" ON kafka
  BEGINSELECT   Mean(accelerometers0) as mean_accelerometers0,
             Mean(accelerometers1) as mean_accelerometers1
    INTO     "kafka.one_year"."mtm1m3-accelerometerdata"
    FROM     "kafka.autogen"."mtm1m3-accelerometerdata"
    GROUP BY time(30s)
  END



Lessons Learned
===============

InfluxDB Retention Policy
-------------------------

InfluxDB HTTP API
-----------------
Connecting to the InfluxDB HTTP API.

Set `max_row_limit=0` in the InfluxDB configuration to avoid data truncation.


References
==========


.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
