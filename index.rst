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

.. Uncomment the line below to use numbered section headings.
  .. sectnum::

.. TODO: Delete the note below before merging new content to the master branch.

.. note::

   **This technote is not yet published.**

This TechNote details the design concepts for the SV-distiller package.
SV-distiller is a redesigned update to validate_drp, with the goal of making it easier for users to implement new metric calculations and easily configure the metrics that will be measured in any given run.

.. Add content here.
.. Do not include the document title (it's automatically added from metadata.yaml).

Overview
========

The SV-distiller is a proposed framework to automate science verification and validation analysis tasks. 
It is intended to be general enough to support evaluation of the data products from both the Alert Production and Data Release Production pipelines, and to be capable of running on precursor datasets, simulations, and LSST on-sky data.
Link to the `repo`_.

.. _repo: https://github.com/lsst/sv-distiller

Design Principles
=================

The overarching design principle driving SV-distiller is to make implementation and measurement of new metrics relatively simple, and to enable configuration-driven measurement of the metrics so that users can calculate only those of immediate interest instead of the entire set, if desired.
A core principle is make the individual analysis component modular to (1) enable scaling to a large dataset that might require many parallel analysis jobs, (2) to facilitate collaborative development, (3) and to make it easy to add new metrics.
SV-distiller assumes that the data reduction has all been performed prior to the validation/verification work.
Thus the starting point is a Butler repository of processed data products.
This enables a user to run SV-distiller on a dataset multiple times, and to do so quickly.
[Note that we haven’t yet decided what the “known point” to which data should be processed will be. But a dataset that has been processed through coaddition and multi-frame/forced source photometry will almost certainly be sufficient.]

SV-distiller will be called by specifying the following:

- A Butler repository containing data processed up to some known point,

- A list of dataIds specifying specific visits/tracts/patches/etc to include in the metric measurement, and

- A configuration file specifying the type of dataset to generate from the input repository, the metrics to measure, and the aggregators that will be used to roll up the measured metrics.

SV-distiller will report the measurements, but will not assess them relative to some specifications (as in, e.g., `lsst.verify`).
However, because the Measurements (and many extras) will be persisted, it will be trivial to assess them relative to specifications, if desired.

Intended to work with lsst.verify `Measurements`_.

.. _Measurements: https://pipelines.lsst.io/py-api/lsst.verify.Measurement.html 

All of the metrics are evaluated independently.
In some cases, closely related metrics (e.g., median of a distribution and number of items from the same distribution above some threshold) may call the same underlying code.

.. figure:: /_static/SV-Distill-crop.png
  :name: block-diagram

  Block diagram showing the high level design that will be described in more detail in following sections.

Objects
=======

We provide an overview of core base classes below.

DataID Generators
-----------------

Data identifier generators will produce an iterable where each element stores the data identifiers necessary for the dataset generators to operate on.
Since the data identifier generators need to know what dataset generator they are serving, it may be that both pieces of functionality will be implemented in the same class.


Dataset Generators
------------------

Dataset generators are responsible for producing the dataset necessary for a measurement algorithm to do its work.
The expected output from the dataset generator is a tabular object, e.g. an astropy.Table.
In many cases, this is a matched catalog of some sort.
Dataset generators will be constructed from a Butler, though other arguments may be passed in via *args and **kwargs.
Possible dataset types include a single visit image, a deep coadd (or set of coadds), a set of visits from which a matched catalog of sources will be created, and possibly even alert packets, images, or metadata (basically, anything accessible by the Butler).

Measurers
---------

The Measurers are the primary objects for evaluating performance metrics given tabular input data.
The output of a Measurer is an lsst-verify Measurement.
In practice, the Measurers can operate on fine-grained data (e.g., an individual visit, individual patch).
It is expected that some of the Measurers will use the extras that can be associated with a Measurement, in addition to the scalar value.

The output lsst-verify Measurements are planned to be stored in an intermediate database to hold the results of metrics computed on fine-grained input data.
This intermediate database should make it possible for users to search for specific units of input data and thereby more easily diagnose anomalies, etc.

Aggregators
-----------

The aggregators compile summary statistics for the dataset of interest, encapsulated as a lsst.verify Measurements.
These high-level metrics can be used to more readily verify pass/fail status, provide a high-level summary of performance, and reveal the presence of outliers in the performance distribution.
The aggregators take as input the lsst.verify Measurements that have been computed by the Measurers.
In the simplest cases, the aggregator may simply pass a Measurement to the next stage of analysis.
In more general cases, an aggregator will compute statistics such as median, mean, maximum, percentiles, evaluating intercepts or thresholds, number of instances above or below some value.

Key concepts
============

High-level Metrics
------------------

This design specifically takes into account the desire to have high-level measurements produced from a set of measurements made on a finer grained scale.
Measurement of these metrics requires defining three key pieces of information:

- The dataset generator class to use for passing data identifiers to the fine grained measurement algorithm.

- The fine grained measurement algorithm to use on each set of data identifiers.

- The aggregator to apply to the list of fine grained measurements.

Registries for Classes/Functions
--------------------------------

A key principle for making this framework function is the concept of registries for each of the main kinds of classes and functions.
In essence these are nothing more than dictionaries of name/object pairs.  This does several things:

- Given a name of one of these objects and a git SHA1, one can tell exactly what code was run at a given time.

- Short names as keys allow us to change our mind about how a measurement is calculated without changing configuration.

- This helps make the definitions of high-level measurements succinct.

Running
=======

A pseudo code is `here`_. 

.. _here: https://github.com/lsst/sv-distiller/blob/initial_stubs/code_design/runner_pseudo.py

The conceptual workflow is to 

#. Use a DataID Generator to create a list of dataid lists. These dataid lists specify the individual units of data for fine-grained analysis.

#. Loop over dataid lists. For each list of dataids, there will be a list of dataset generators. Create the associated dataset.

#. Loop of datasets. For each, there will be a list of associated Measurers. 

#. Run the associated Measurers and push the output Measurements to intermediate database.

#. Loop over all of the high-level metrics. For each, gather the associated intermediate results and compute summary statistics using the associated aggregator.

#. Optionally, run an afterburner script on the set of output high-level metrics to evaluate which specifications have been met.

Plans for Code Development
==========================

Identify the set of DataSetGenerators, Measurers, and Aggregators that are needed. This step is building the registry of classes / functions.

.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :style: lsst_aa
