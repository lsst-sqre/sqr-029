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
* Total memory: 22.50 GB

The above specs are sufficient for running JVM, but not recommended for production. See `Running Kafka in Production <https://docs.confluent.io/current/kafka/deployment.html>`_  and `InfluxDB hardware guidelines for single node <https://docs.influxdata.com/influxdb/v1.7/guides/hardware_sizing/#general-hardware-guidelines-for-a-single-node>`_ for more information.

The performance requirements for InfluxDB, based on the expected throughout (see below), falls into the `moderate load <https://docs.influxdata.com/influxdb/v1.7/guides/hardware_sizing/#general-hardware-guidelines-for-a-single-node>`_  category. Thus a single InfluxDB node instance for the DM-EFD should be enough.

Terraform and Helm
------------------

Monitoring
----------

Connecting Kafka and InfluxDB
=============================

At the time of this implementation, the `Confluent InfluxDB connector <https://docs.confluent.io/current/connect/kafka-connect-influxdb/index.html>`_ was still in preview and did not have the functionality we needed. Instead of the Confluent InfluxDB connector, we used the `InfluxDB Sink connector developed by Landoop <https://docs.lenses.io/connectors/sink/influx.html>`_.

We added the Landoop InfluxDB Sink connector plugin version 1.1 to the ``cp-kafka-connect`` container and implemented scripts to facilitate its configuration.

A limitation of version 1.1, though, was the lack of support for the Avro ``array`` data type, which was solved by `contributing to the plugin development <https://github.com/Landoop/stream-reactor/pull/522>`_.


The InfluxDB schema
-------------------

One of the characteristics of InfluxDB is that it creates the database schema when it writes the data to the database, this is commonly known as *schemaless* or *schema-on-write*. The advantage is that no schema creation and database migrations are needed, greatly simplifying the database management. However,  it also means that it is not possible to enforce a schema with InfluxDB only.

In the proposed architecture, the schema is controlled by Kafka through `Avro and the Schema Registry <https://docs.confluent.io/current/schema-registry/docs/index.html#schemaregistry-intro>`_. As the schema may need to evolve, it is important for InfluxDB, and for other consumers, to be able to handle data encoded with both old and new schema seamlessly. While this report does not explore `schema evolution <https://docs.confluent.io/current/schema-registry/docs/avro.html#data-serialization-and-evolution>`_  that is undoubtedly important and we will revisit.

The data in InfluxDB, however, does not necessarily need to follow the Avro schema. The InfluxDB Sink Connector supports `KCQL <https://docs.lenses.io/connectors/sink/influx.html#kcql-support>`_, the Kafka Connect Query Language, that can be used to select fields to define the target measurement, and `set tags to annotate the measurements <https://docs.influxdata.com/influxdb/v1.7/concepts/schema_and_data_layout/>`_.

In the current implementation, the InfluxDB schema is the simplest possible. We create an InfluxDB measurement with the same name as the topic and select all fields from the topic.

Example of an Avro schema for the ``MTM1M3_accelerometerData`` SAL topic, and the corresponding InfluxDB schema:

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
  2. InfluxDB does not support ``array`` data type. Fields named like ``<field name>0, <field name>1, ...`` were extracted from arrays in the Avro message.


Visualizing SAL Topics with Chronograf
--------------------------------------

`Chronograf <https://chronograf-demo.lsst.codes>`_ presents the SAL topics as InfluxDB measurements. One can use the Explore tool to browse and visualize them.


.. figure:: /_static/chronograf.png
   :name: Chronograf Explore tool.
   :target: _static/chronograf.png

   Visualization using the Chronograf Explore tool.

For monitoring the different telescope and observatory subsystems, it is possible to organize these visualizations in Dashboards.


The SAL mock experiment
=======================

With the SAL mock experiment, we want to access the performance of our prototype implementation of the DM-EFD.

In the following sections we explain the experiment we designed, how we produced messages for the SAL topics, and finally, we characterize the mean latency for a message from the time it was produced to the time InfluxDB writes it to the disk. Finally, we measure the InfluxDB throughput during the experiment.


Designing the experiment
------------------------

To run a realistic experiment that emulates the EFD, besides producing messages for each SAL topic, one would need to know the frequency of every topic, which is not available in the SAL schema.

From the current SAL XML schema we have a total of 1051 topics, in which 274 are commands, 541 are log events, and 236 are telemetry. For simplicity, we assume a distribution of frequencies for the different types of topics, as shown in the table below.

============ ================= ============ =============== ===================================
Producer ID  Topic type        # of topics  Frequency (Hz)  Expected throughput (messages/s)
============ ================= ============ =============== ===================================
`0`_         SAL Commands      274          1               274
`1`_         SAL Log Events    541          10              5410
`2`_         SAL Telemetry     236          100             23600
============ ================= ============ =============== ===================================

.. _`0`: https://github.com/lsst-sqre/kafka-efd-demo/blob/tickets/DM-17052/k8s-apps/salmock-1node-commands-1hz.yaml

.. _`1`: https://github.com/lsst-sqre/kafka-efd-demo/blob/tickets/DM-17052/k8s-apps/salmock-1node-logevents-10hz.yaml

.. _`2`: https://github.com/lsst-sqre/kafka-efd-demo/blob/tickets/DM-17052/k8s-apps/salmock-1node-logevents-10hz.yaml

- Total number of topics: 1051
- Total expected throughput: 29284 messages/s
- Experiment Duration: 16h

Producing SAL topics
--------------------

- Converting SAL XML schema to Apache Avro
- The AIOKafkaProducer

The measured throughput
^^^^^^^^^^^^^^^^^^^^^^^

.. figure:: /_static/salmock_produced_total.png
   :name: Producer metric.
   :target: _static/salmock_produced_total.png

   The figure shows the producer throughput measured by the ``salmock_produced_total`` Prometheus metric.

- Number of topics produced: 1051
- Maximum measured throughput for the producers: 1330 messages/s

Another Prometheus metric of interest is ``cp_kafka_server_brokertopicmetrics_bytesinpersec`` which give us a mean throughput at the brokers, for all topics, of 40KB/s. We observe the same value when looking at the Network traffic as monitored by the InfluxDB telegraf client.

As a point of comparison, this throughput is lower than the *Long-term mean ingest rate to the Engineering and Facilities Database of non-science images required to be supported* for the EFD of 1.9 MB/s from **OCS-REQ-0048**.

We can do better by improving the producer throughput, and we demonstrate that we can reach a higher performance with a simple test when accessing the InfluxDB maximum throughput for the current setup.

Latency measurements
--------------------

.. figure:: /_static/latency.png
   :name: Roundtrip latency for a telemetry message.
   :target: _static/latency.png

   The figure shows the roundtrip latency for a telemetry topic during the experiment, measured as the difference between the producer and consumer timestamps.

We characterize the roundtrip latency as the difference between the time the message was produced and the time InfluxDB writes it to the disk.

**The median roundtrip latency for a telemetry topic produced over the duration of the experiment was 183ms with 99% of the messages with latency smaller than 1.34s.**

This result would allow for quasi-realtime access to the telemetry stream from resources at the LDF.  That would not be possible with the current baseline design (see discussion in `DMTN-082 <https://dmtn-082.lsst.io/>`_).


The InfluxDB throughput
-----------------------

.. figure:: /_static/influxdb.png
   :name: InfluxDB throughput.
   :target: _static/influxdb.png

   The figure shows the InfluxDB throughput in units of points per minute.

Because of the current InfluxDB schema, an InfluxDB point is equivalent to a message. The measured InfluxDB throughput during the experiment was ~80k points/min or 1333 messages/s, which is the producer throughput (see above). This result is supported by the very low latency observed.

InfluxDB provides a metric ``write_error`` that counts the number of errors when writing points, and it was ``write_error=0`` during the whole experiment.

During the experiment, we also saw the InfluxDB disk filling up at a rate of 682MB/h or 16GB/day. Even with `InfluxDB data compression <https://www.influxdata.com/blog/influxdb-0-9-3-released-with-compression-improved-write-throughput-and-migration-from-0-8/>`_ that means 5.7TB/year which seems too much, especially if we want to query over longer periods like **OCS-REQ-0047** suggests, e.g., *"raft 13 temperatures for past two years"*. For the DM-EFD, we are considering downsampling the time series and using a retention policy, as discussed in the `Lessons Learned`_.

Finally, a simple test can be done to assess the maximum InfluxDB throughput for the current setup.

We stopped the InfluxDB Sink connector, and let the producer running during a period T. The Kafka brokers cached the messages produced during T, and as soon as the connector was re-started, all the messages were flushed to InfluxDB as if they were produced in a much higher throughput.

The figure below shows the result of this test, where we see a measured throughput of 1M points per minute or 16k messages per second, a factor of 12 better than the previous result. Also, we had ``write_error=0`` during this test.


.. figure:: /_static/influxdb_max.png
   :name: InfluxDB maximum throughput.
   :target: _static/influxdb_max.png

   InfluxDB maximum throughput measured as the number of points/min.

In particular, these results are very encouraging because both Kafka and InfluxDB were deployed in modest hardware, and with default configurations. There is indeed room for improvement, and many aspects to explore in both Kafa and InfluxDB deployments.

The SAL Kafka writer
====================


Lessons Learned
===============

Downsampling and data retention
-------------------------------

It was evident during the experiments that the disks fill up pretty quickly. The influxDB disk was filling up at a rate of ~700M/h which means that the 128G storage would be filled up in ~7 days. Similarly, for Kafka, we filled up the 5G disk of each broker in a few days. That means we need downsampling the data if we don't want to lose it and configure retention policies to discard data after it is no longer useful automatically.

In InfluxDB it is easy to configure both `downsampling and data retention <https://docs.influxdata.com/influxdb/v1.7/guides/downsampling_and_retention/>`_.

InfluxDB organizes time series data in *shards* and will drop an entire shard when it enforces the retention policy. That means the retention policy's duration must be longer than the shard duration.

For the experiments, we have created a `Kafka` database in InfluxDB to have a default retention policy of 24h and shard duration of 1h following the `retention policy documentation <https://docs.influxdata.com/influxdb/v1.7/query_language/database_management/#create-retention-policies-with-create-retention-policy>`_.

InfluxDB creates retention policies per database, and it is possible to have multiple retention policies for the same database. To preserve data for a more extended period, we have created another retention policy with a duration of 1 year and a `Continuous Query <https://docs.influxdata.com/influxdb/v1.7/query_language/continuous_queries/>`_ to average the time series every 30s.


.. figure:: /_static/downsampling.png
   :name: Downsampling a time series using a continuous query.
   :target: _static/downsampling.png

Example of a continuous query for the `mtm1m3-accelerometerdata` topic. If we produce topics at 100Hz and average the time series in intervals of 30 seconds, the downsampling factor is 30000.

::

  CREATE continuous query "mtm1m3-accelerometerdata" ON kafka
  BEGINSELECT   Mean(accelerometers0) as mean_accelerometers0,
             Mean(accelerometers1) as mean_accelerometers1
    INTO     "kafka.one_year"."mtm1m3-accelerometerdata"
    FROM     "kafka.autogen"."mtm1m3-accelerometerdata"
    GROUP BY time(30s)
  END


The retention policy of 24h in InfluxDB suggests that we configure a Kafka retention policy for the logs and topic offsets with the same duration. It means that InfluxDB can be unavailable for 24h and still recover the messages from the Kafka brokers. We added the following `configuration parameters <https://kafka.apache.org/documentation/#configuration>`_ to the ``cp-kafka`` helm chart:


::

  ## Kafka Server properties
  ## ref: https://kafka.apache.org/documentation/#configuration
  configurationOverrides:
    offsets.retention.minutes: 1440
    log.retention.hours: 24


The InfluxDB HTTP API
---------------------

InfluxDB provides an HTTP API for accessing the data when using the HTTP API we
set ``max_row_limit=0`` in the InfluxDB configuration to avoid data truncation.

A code snippet to retrieve data from a particular topic would look like:

::

  import requests

  INFLUXDB_API_URL = "https://kafka-influxdb-demo.lsst.codes"
  INFLUXDB_DATABASE = "kafka"

  def get_topic_data(topic):
    params={'q': 'SELECT * FROM "{}\"."autogen"."{}" where time > now()-24h'.format(INFLUXDB_DATABASE, topic)}
    r = requests.post(url=INFLUXDB_API_URL + "/query", params=params)

    return r.json()


Backing up an InfluxDB database
--------------------------------

InfluxDB supports `backup and restores <https://docs.influxdata.com/influxdb/v1.7/administration/backup_and_restore/>`_ functions on online databases. A backup of a 24h worth of data database took less than 10 minutes in our current setup while running the SAL Mock Experiment and ingesting data at 80k points/min.

Backup files are split by shards, in `Downsampling and data retention`_ we configured our retention policy to 24h and shard duration to 1h, so the resulting backup has 24 files.

We do observe a drop in the ingestion rate to 50k points/min during the backup, but no write errors and Kafka design ensures nothing gets lost even if the InfluxDB ingestion rate slows down.


.. figure:: /_static/influxdb_backup.png
   :name: Drop in the ingestion rate during a backup of the DM-EFD database.
   :target: _static/influxdb_backup.png




User Defined Functions
----------------------

APPENDIX
========

Kafka Terminology
-----------------

- Each server in the Kafka clusters is called a **broker**.
- Kafka stores messages in a category name called **topic**.
- A Kafka message is a key-value pair, and the key, message, or both, can be serialized as **Avro**.
- A **schema** defines the structure of the Avro data format.
- The Schema Registry defines a **subject** as a scope where a schema can evolve. The name of the subject depends on the configured subject name strategy, which by default is set to derive the subject name from the topic name.
- The processes which publish messages to Kafka are called **producers**. Also, it publishes data on specific topics.
- **Consumers** are the processes that subscribe to topics.
- The position of the consumer in the log is called **offset**. Kafka retains that on a per-consumer basis.
- The Kafka **connector** permits to build and run reusable consumers or producers that connects existing applications to Kafka topics.


InfluxDB Terminology
--------------------

- A **measurement** is conceptually similar to an SQL table. The measurement name describes the data stored in the associated fields.
- A **field** corresponds to the actual data and are not indexed.
- A **tag** is used to annotate your data  (metadata) and is automatically indexed.
- A **point** contains the field-set of a series for a given tag-set and timestamp. Points are equivalent to messages in Kafka.
- A measurement and a tag-set define a **series**. A *series** contains points.
- The **series cardinality** depends mostly on how the tag-set is designed. A rule of thumb for InfluxDB is to have fewer series with more points than more series with fewer points to improve performance.
- A **database** store one or more series.
- A database can have one or more **retention policies**.

References
==========

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
