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

The GKE cluster specs
---------------------

* Machine type: n1-standard-2 (2 vCPUs, 7.5 GB memory)
* Size: 3 nodes
* Total cores: 6 vCPUs
* Total memory:	22.50 GB

The above specs are sufficient for JVM, but not recommended for production. See `Running Kafka in Production <https://docs.confluent.io/current/kafka/deployment.html>`_  and `InfluxDB hardware guidelines for single node <https://docs.influxdata.com/influxdb/v1.7/guides/hardware_sizing/#general-hardware-guidelines-for-a-single-node>`_ for more information.

The performance requirements for InfluxDB, based on the expected throughout (see below), falls into the `moderate load <https://docs.influxdata.com/influxdb/v1.7/guides/hardware_sizing/#general-hardware-guidelines-for-a-single-node>`_  category, thus a single InfluxDB node instance should be enough.

Terraform and Helm
------------------

Monitoring
----------

The SAL Kafka writer
====================

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

A more realistic experiment would require knowing the frequency of each topic which is not always available in the SAL schema. We assume that the distribution of frequencies above, and the total expected throughput, can be used as an upper limit.

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

At the time of this implementation, the `Confluent InfluxDB connector <https://docs.confluent.io/current/connect/kafka-connect-influxdb/index.html>`_ was still in preview and did not have the functionality we needed. Instead of the Confluent InfluxDB connector, we used the `InfluxDB Sink connector developed by Landoop <https://docs.lenses.io/connectors/sink/influx.html>`_.

We added the Landoop InfluxDB Sink connector 1.1 plugin to the ``cp-kafka-connect`` container, and implemented scripts to facilitate its configuration.

A limitation of version 1.1, though, was the lack of support for the Avro ``array`` data type, which was solved by `contributing to the plugin development <https://github.com/Landoop/stream-reactor/pull/522>`_.



The InfluxDB schema
^^^^^^^^^^^^^^^^^^^

One of the characteristics of InfluxDB is that the schema is created as the data is written to the database, this is commonly referred as *schemaless* or *schema-on-write*. The advantage is that no schema creation and database migrations are needed simplifying the database management, but it also means that it is not possible to enforce a schema with InfluxDB only.

In the proposed architecture, the schema is controlled by Kafka through `Avro and the Schema Registry <https://docs.confluent.io/current/schema-registry/docs/index.html#schemaregistry-intro>`_. As the schema may need to evolve over time it is important for InfluxDB, and for other consumers, to be able to handle data encoded with both old and new schema seamlessly. While `schema evolution <https://docs.confluent.io/current/schema-registry/docs/avro.html#data-serialization-and-evolution>`_ is not explored throughout the SAL Mock experiment it is certainly important and must be revisited.

The data written to InfluxDB, however, does not necessarily need to follow exactly the Avro schema. The InfluxDB Sink Connector supports `KCQL <https://docs.lenses.io/connectors/sink/influx.html#kcql-support>`_, the Kafka Connect Query Language that can be used to select fields, define the target measurement, and `set tags to annotate the measurements <https://docs.influxdata.com/influxdb/v1.7/concepts/schema_and_data_layout/>`_.

In the current implementation, the InfluxDB schema is the simplest possible. We create an InfluxDB measurement with the topic name and select all fields from the topic.

Example of Avro schema for the ``MTM1M3_accelerometerData`` topic, and the corresponding InfluxDB schema:

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

   The roundtrip latency for a telemetry topic during the experiment, measured as the difference between the producer and InfluxDB (consumer) timestamps.

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

An InfluxDB database stores points. In the current data model a point has a timestamp, a measurement, and fields. Thus, by construction an InfluxDB point is equivalent to an message.

The measured InfluxDB throughput during the experiment was ~80k points/min or ~1.3k messages/s, which is basically determined by the producer throughput (see above).

InfluxDB provides a metric ``write_error`` that counts the number of errors when writing points, and it was ``write_error=0`` during the whole experiment.

However, the measured producer throughput is lower than expected throughput. We could improve the performance of the producer code or put more resources to run the producer jobs, but a simple test can be done to assess the maximum InfluxDB throughput.

We stopped the InfluxDB Sink connector, and let the producer to run for a period of time T, the messages produced during T were cached at the Kafka brokers. As soon as the connector was res-started, all the messages were flushed to InfluxDB as if they were produced in a much higher throughput.

The result of this test is shown in the  figure below, were we see a measured throughput of 1M points/min (or 16k messages/s) about 10 times higher than the previous result.


.. figure:: /_static/influxdb_max.png
   :name: InfluxDB maximum throughput.
   :target: _static/influxdb_max.png

   InfluxDB maximum throughput measured as number of points/min.

Also, ``write_error=0`` during this test, and we note again that we are running on modest hardware and using the InfluxDB default configuration.



Lessons Learned
===============

Downsampling and data retention
-------------------------------

It was clear during the experiments that the disks fill up pretty quickly. InfluxDB was writing at a rate of ~700M/h which means that the 128G disk would be filled up in ~7 days. Similarly, for Kafka, we filled up the 5G disk of each broker in a few days. That means we need downsampling the data if we don't want to loose it and configure retention policies to automatically discard data after it's no longer useful.

Both `downsampling and data retention <https://docs.influxdata.com/influxdb/v1.7/guides/downsampling_and_retention/>`_ can be easily configured in InfluxDB.

Time Series data is organized in *shards*, and InfluxDB will drop an entire shard when the retention policy is enforced. That means the retention policy's duration must be longer than the shard duration.

For the experiments, we have created our `kafka` database in InfluxDB to have a default retention policy of 24h and and shard duration of 1h following the `retention policy documentation <https://docs.influxdata.com/influxdb/v1.7/query_language/database_management/#create-retention-policies-with-create-retention-policy>`_.

Retention policies are created per database, and it is possible to have multiple retention policies for the same database. In order to preserve data for a longer time period, we have created another retention policy with a duration of 1 year and a `Continuos Query <https://docs.influxdata.com/influxdb/v1.7/query_language/continuous_queries/>`_ to average the time series every 30s.


.. figure:: /_static/downsampling.png
   :name: Downsampling a time series using a continuous query.
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

The retention policy of 24h in InfluxDB suggests that we configure a Kafka retention policy for the logs and topic offsets with the same duration. That means InfluxDB could be unavailable for 24h and still recover the messages from the Kafka brokers. The following `configuration <https://kafka.apache.org/documentation/#configuration>`_ parameters were added to the Kafka helm chart:


::

  offsets.retention.minutes: 1440
  log.retention.hours: 24


InfluxDB HTTP API
-----------------
InfluxDB provides an HTTP API for accessing the data, when using the HTTP API we
set ``max_row_limit=0`` in the InfluxDB configuration to avoid data truncation.


APPENDIX
========

Kafka Terminology
-----------------

- Each server in the Kafka clusters is called a **broker**.
- Kafka messages are stored as well as published in a category name called **topic**.
- A kafka message is a key-value pair, and the key, message, or both, can be serialized as **Avro**.
- A **schema** defines the structure of the Avro data format.
- A **subject** is defined in the Schema Registry as a scope where a schema can evolve. The name of the subject depends on the configured subject name strategy, which by default is set to derive subject name from topic name.
- The processes which publish messages to Kafka are called **producers**. In addition, it publishes data on specific topics.
- The processes that subscribe to topics are called **consumers**.
- The position of the consumer in the log and which is retained on a per-consumer basis is called **offset**.
- The Kafka **connector** permits to build and run reusable consumers or producers that connects existing applications to Kafka topics.


InfluxDB Terminology
--------------------

- A **measurement** is conceptually similar to an SQL table. The measurement name describes the data stored in the associated fields.
- A **field** corresponds to the actual data and are not indexed.
- A **tag** is used to annotate your data  (metadata) and is automatically indexed.
- A **point** contains the field-set of a series for a given tag-set and timestamp. Points are equivalent to messages in Kafka.
- A **series** contains Points and is defined by a measurement and a tag-set.
- The **series cardinality** depends essentially on how the tag-set is designed. A rule of thumb for InfluxDB is to have fewer series with more points than more series with fewer points to improve performance.
- A **database** store one or more series.
- A database can have one or more **retention policies**.





References
==========


.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
