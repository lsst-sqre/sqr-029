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

The Confluent Kafka Platform
----------------------------

The InfluxData stack
--------------------

Why a time series database?
^^^^^^^^^^^^^^^^^^^^^^^^^^^


Why InfluxData?
^^^^^^^^^^^^^^^

Deploying Confluent Kafka and the InfluxData Stack
==================================================

The GKE Cluster Specification
-----------------------------

* Machine type: n1-standard-2 (2 vCPUs, 7.5 GB memory)
* Size: 3 nodes
* Total cores: 6 vCPUs
* Total memory:	22.50 GB

Monitoring
----------

The SAL Mock experiment
=======================

Designing the experiment
------------------------

============= ============ =============== ===================================
Message Type  # of topics  Frequency (Hz)  Expected throughput (# messages/s)
============= ============ =============== ===================================
SAL Command   274           1              274
SAL Log Event 541           10             5410
SAL Telemetry 236           100            23600
============= ============ =============== ===================================

Total expected throughput (# messages/s): 29284

Experiment Duration: 16h


Producing SAL Topics
--------------------

Converting SAL XML Schema to Apache Avro
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

AIOKafkaProducer
^^^^^^^^^^^^^^^^

The measured throughput
^^^^^^^^^^^^^^^^^^^^^^^

.. figure:: /_static/salmock_produced_total.png
   :name: Producer metric.

   The producer throughput as measured by the `salmock_produced_total` metric.


Peak measured throughput (# messages/s): 1330

Connecting Kafka and InfluxDB
-----------------------------

The Landoop InfluxDB Sink connector
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Connector Configuration.

Supporting Avro arrays and bytes data types.

The InfluxDB datamodel
^^^^^^^^^^^^^^^^^^^^^^

Latency measurements
^^^^^^^^^^^^^^^^^^^^

.. figure:: /_static/latency.png
   :name: Roundtrip latency for a telemetry message.

   The roundtrip latency for a telemetry topic produced at 100MHz measured as the difference of the InfluxDB and Kafka timestamps.

The median latency over the experiment duration is 183ms with 99% of the messages < 1.34s.


The InfluxDB throughput
^^^^^^^^^^^^^^^^^^^^^^^

.. figure:: /_static/influxdb.png
   :name: InfluxDB throughput.

   InfluxDB throughput measured as number of points per minute.



Configuring a Retention Policy
------------------------------

Downsampling Time Series
------------------------

.. figure:: /_static/downsampling.png
   :name: Downsampling a SAL Telemetry message using a continuous query.


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
