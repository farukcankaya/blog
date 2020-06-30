---
toc: true
layout: post
description: Exploring new dataset format options for OpenML.org
categories: [OpenML, Data]
comments: true
title: Finding a standard dataset format for machine learning
image: images/fastpages_posts/format.jpg
author: Pieter Gijsbers, Mitar Milutinovic, Prabhant Singh, Joaquin Vanschoren
---

With OpenML, we aim to take a worry-free, &#39;zen&#39;-like approach to working with machine learning datasets, making them easy and reliable to use. We want to offer training data that can be easily (or automatically) used. As such, you can now download all OpenML datasets in the same format, with the same rich meta-data, and directly load it and start building models without manual intervention. For historical reasons, we have done this by internally storing all data in the [ARFF](https://www.cs.waikato.ac.nz/ml/weka/arff.html) data format, which is a simple single-table format that includes meta-data such as the correct feature data types.

However, this format is currently holding us back: it is not ideal for storing large datasets, the format is only loosely defined causing different parsers to behave differently, and many new machine learning tasks require multi-table data. For instance, image segmentation or object detection tasks have both images and varying amounts of annotations per image.

Hence, we need a better data format to *internally* store machine learning datasets in the foreseeable future. This blog post presents out process and insights. We would love to hear your thoughts and experiences before we make any decision on how to move forward. 

**Scope**

We first define the general scope of the usage of the format:

- We mainly need a format that is **useful for data storage and transmission**. We can always convert data during upload or download in OpenML&#39;s client APIs. For instance, people may upload a Python pandas dataframe to OpenML, and later get the same dataframe back, without realizing or caring how the data was stored in the meantime. If people want to store the data locally, they can download it in the format they like (e.g. a memory-mapped format like Arrow/Feather for fast reading or TFRecords for people using TensorFlow). Additional code can facilitate such conversions.
- To make the data easy to use, there should be a **standard way to represent specific types of data** (i.e. a fixed schema that can be verified), so that we can easily read and store datasets in a uniform way. For instance, all tabular data should be stored in a uniform way. Storing datasets in different formats is of course easier, but would require dataset-specific code for loading, which itself requires maintenance, and it will be harder to check quality and extract meta-data.
- The format should **allow storing most machine learning datasets,** including images, video, audio, text, graphs, and multi-tabular data such as object recognition tasks and relational data. Data such as images can be converted to numeric formats (e.g. pixel values) for storage.

**Impact on OpenML (simplicity, maintenance)**

Since OpenML is a community project, we want to keep it as easy as possible to use and maintain:

- We prefer a **single internal data format** rather than multiple ones. The latter would impose more maintenance on both server-side and client-side.
- we need to be able to identify dataset types so that we can roll out support for different types gradually. We can't support arbitrary data files.
- **We will require machine-readable schemas** (in a specific language) that describe how a certain &#39;type&#39; data is formatted. Examples would be a schema for tabular data, a schema for annotated image data, etc. Every dataset should specify the schema it satisfies, and we should be able to **validate** whether the dataset indeed satisfies the schema.
- OpenML would have a set of **allowed schemas**. We will stick to tabular data until other schemas are defined, and we can validate and handle those datasets.
- When no agreed upon schema exists, we could offer a forum for the community to discuss and agree on a standard schema, in collaboration with other initiatives (e.g. [frictionlessdata](https://frictionlessdata.io/)). [For instance, new schemas could be created in a github repo to allow people to do create pull requests. They could be effectively used once they are merged.]

**Requirements**

To draw up a shortlist of data formats, we used the following requirements:

- Maintenance. We are still a small community so we would prefer a format which is stable and fully maintained over something which we would have to maintain ourselves.
- Support for the format in various programming languages, including well-maintained and stable libraries.
- Support for reading the data without copying everything into memory (e.g. incremental reads/writes), and for subselecting parts of the data and operating only on that.
- Version control, some way to see what is different between different versions of a dataset.
- Ideally, there is a way to detect bitflip errors during storage or transmission.
- The dataset format should support multiple &quot;resources&quot; to support cases when we would like to store collections of files or multiple relational tables.
- Support for storing binary blobs and vectors of different lengths.
- Anything else being equal, the dataset format should be compact.
- Ideally, there should be support for storing sparse data.
- A nice to have is that we can store some meta-data inside the file. OpenML can generate more extensive meta-data on the fly, but storing a minimal set inside the file may be useful.

**Shortlist**

We decided to investigate the following formats in more detail:

[**Arrow**](https://arrow.apache.org/) **/** [**Feather**](https://github.com/wesm/feather)

Benefits:

- Great for locally caching files after download
- Memory-mapped, so very fast reads

Drawbacks:

- Not stable enough yet and not ideal for long-term storage. The authors also discourage it for long-term storage.
- Limited to one data structure per file, but that data structure can be complex (e.g. dict).

[**Parquet**](https://parquet.apache.org/)

Benefits:

- Used in industry a lot, especially in combination with Apache Spark. Good community of practice
- Well-supported and maintained, but not all Parquet features are supported in every library. E.g. the python library does not support partial read/writes.
- Simple structure
- Built-in compression (columnar storage), very efficient long-term data storage

Drawbacks:

- The different parsers (e.g. [Parquet support inside Arrow](https://arrow.apache.org/docs/python/parquet.html), [fastparquet](https://fastparquet.readthedocs.io/)) implement different parts of the Parquet format (and different set of compression algorithms), meaning that the output may not be [compatible](https://fastparquet.readthedocs.io/en/latest/#caveats-known-issues) [with](https://kb.databricks.com/data/wrong-schema-in-files.html) other parsers (in other languages).
- Support limited to single-table storage. There is good support to convert to and from pandas DataFrames, but there is less support for more complex data structures. Also, Parquet files created by other libraries might not be readable into pandas. More complicated data schemas might be possible, but are not supported in Python. For instance, there doesn&#39;t seem to be an apparent way to store an object detection dataset (with images and annotations) as a single parquet file.
- Limited support for incremental reading/writing. None of the parsers we looked at (e.g. [Parquet support inside Arrow](https://arrow.apache.org/docs/python/parquet.html), [fastparquet](https://fastparquet.readthedocs.io/)) allows incremental writes, which may be an issue for large datasets when we (or end users) cannot load the data into memory. We can easily store large datasets in multiple (partitioned) parquet files, but that brings additional complexity in using them.

[**SQLite**](https://www.sqlite.org/index.html)

Benefits:

- SQLite was the easiest to use. It was comparably fast to HDF5 in our tests.
- Very good support in all languages. For instance, it is [built-in](https://docs.python.org/3.7/library/sqlite3.html) in Python.
- Very flexible access to parts of the data. SQL queries can be used to select any subset of the data.

Drawback:

- It supports only 2000 columns, and we have quite a few datasets with more than 2000 features. Hence, storing large tabular data will require mapping data differently, which would add a lot of additional complexity.
- Writing SQL queries requires knowledge of the internal data structure (tables, keys,...).

[**HDF5**](https://www.hdfgroup.org/solutions/hdf5/)

Benefits:

- Very good support in all languages. Has well-tested parsers, all using the same C implementation.
- Widely accepted format in the deep learning community to store data and models.
- Widely accepted format in many scientific domains (e.g. astronomy, bioinformatics,...)
- Provides built-in compression. Constructing and loading datasets was reasonably fast.
- Very flexible. Should allow to store any machine learning dataset as a single file.
- Allows easy inclusion of meta-data inside the file, creating a self-contained file.
- Self-descriptive: the structure of the data can be easily read programmatically. For instance, &#39;h5dump -H -A 0 mydata.hdf5&#39; will give you a lot of detail on the structure of the dataset.

Drawbacks:

- Complexity. We cannot make any a priori assumptions about how the data is structured. We need to define schema and implement code that automatically validates that a dataset follows a specific schema (e.g. using h5dump to see whether it holds a single dataframe that we could load into pandas). We are unaware of any initiatives to define such schema.
- The format has a very long and detailed specification. While parsers exist we don&#39;t really know whether they are fully compatible with each other.

**CSV**

Benefits:

- Very good support in all languages.
- Easy to use, requires very little additional tooling
- Easy versioning with git LFS. Changes in different versions can be observed with a simple git diff.
- The current standard used in [frictionlessdata](https://frictionlessdata.io/).
- There exist schema to express certain types of data in CSV (see [frictionlessdata](https://frictionlessdata.io/)).

Drawbacks:

- Not very efficient for storing floating point numbers
- Not ideal for very, very large datasets (when data does not fit in memory/disk)
- Many different dialects exist. We need to decide on a standardized dialect and enforce that only that dialect is used on OpenML ([https://frictionlessdata.io/specs/csv-dialect/](https://frictionlessdata.io/specs/csv-dialect/)). The dialect specified in [RFC4180](https://tools.ietf.org/html/rfc4180), which uses the comma as delimiter and the quotation mark as quote character, is often recommended.



**Overview**

|   | Parquet | HDF5 | SQLite | CSV |
| --- | --- | --- | --- | --- |
| Consistency across <br>different platforms | ? | ✅ | ✅ | ✅ (dialect) |
| Support and documentation | ✅ | ✅ | ✅ | ✅ |
| Supports very large and high-dimensional datasets | ✅ | ✅ | ❌ (limited nr. columns<br> per table) | ✅ Storing tensors<br> requires flattening. |
| Simplicity | ✅ | ❌ (basically full <br>file system) | ✅ (it's a database) | ✅ |
| Metadata support | Only minimal | ✅ | ✅ | ❌ (requires separate <br> metadata file) |
| Maintenance | Apache project, open<br> and quite [active](https://www.slideshare.net/Hadoop_Summit/the-columnar-roadmap-apache-parquet-and-apache-arrow-102997214) | Closed group,<br> but [active](https://www.slideshare.net/HDFEOS/hdf5-roadmap-20192020) community on <br>Jira and conferences | Run by a [company](https://www.sqlite.org/prosupport.html).<br> Uses an email list. | ✅ |
| Available examples of<br> usage in ML | ✅ | ✅ | ❌ | ✅ |
| Allows incremental <br>reads/writes | Yes, but not <br>supported by current<br> Python libs | ✅ | ✅ | Yes (but not <br>random access) |
| Flexibility | Only tabular<br> data supported | Very flexible, <br>maybe too flexible | Relational multi-table | Only tabular |

**Performance benchmarks**

There exist some prior benchmarks ([here](https://tech.jda.com/efficient-dataframe-storage-with-apache-parquet/) and [here](https://towardsdatascience.com/the-best-format-to-save-pandas-data-414dca023e0d)) on storing dataframes. These only consider single-table datasets. We also ran our own [benchmark](https://gitlab.com/mitar/benchmark-dataset-formats) to compare the writing performance of those data formats for more complex machine learning datasets, such as pedestrian detection and music analysis (note that we could not find a way to store these in one file in Parquet). Reading benchmarks have yet to be done.

**Version control**

Version control for large datasets is tricky. For CSV, we could use [git LFS](https://git-lfs.github.com/) store the datasets and have automated versioning of datasets. We found it quite easy to export all OpenML dataset to GitLab: [https://gitlab.com/data/d/openml](https://gitlab.com/data/d/openml).

The binary formats do not allow us to track changes in the data, only to recover the exact versions of the datasets you want (and their metadata).

**We need your help!**
If we have missed any format we should investigate, or misunderstood those we have investigated, or missed some best practice, please tell us.
You are welcome to comment below, or send us an email at openmlhq@googlegroups.com


**Contributors to this blog post:**

Mitar Milutinovic, Prabhant Singh, Joaquin Vanschoren, Pieter Gijsbers
