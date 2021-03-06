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

Machine learning data is commonly shared in whatever form it comes in (e.g. images, logs, tables) without being able to make strict assumptions on what it contains or how it is formatted. This makes machine learning hard because you need to spend a lot of time figuring out how to parse and deal with it. Some datasets are accompanied with loading scripts, which are language-specific and may break, and some come with their own server to query the dataset. These do help, but are often not available, and still require us to handle every dataset individually.

With OpenML, we aim to take a stress-free, &#39;zen&#39;-like approach to working with machine learning datasets. To make training data easy to use, OpenML serves thousands of datasets in the same format, with the same rich meta-data, so that you can directly load it (e.g. in numpy,pandas,...) and start building models without manual intervention. For instance, you can benchmark algorithms across hundreds of datasets in a simple loop.

For historical reasons, we have done this by internally storing all data in the [ARFF](https://www.cs.waikato.ac.nz/ml/weka/arff.html) data format, a CSV-like text-based format that includes meta-data such as the correct feature data types. However, this format is loosely defined, causing different parsers to behave differently, and the current parsers are memory-inefficient which inhibits the use of large datasets. A more popular format these days is [Parquet](https://parquet.apache.org/), a binary single-table format. However, many current machine learning tasks require multi-table data. For instance, image segmentation or object detection tasks have both images and varying amounts of annotations per image. 

In short, we are looking the best format to *internally* store machine learning datasets in the foreseeable future, to extend OpenML towards all kinds of modern machine learning datasets and serve them in a uniform way. This blog post presents out process and insights. We would love to hear your thoughts and experiences before we make any decision on how to move forward. 

**Scope**

We first define the general scope of the usage of the format:

- It should be **useful for data storage and transmission**. We can always convert data during upload or download in OpenML&#39;s client APIs. For instance, people may upload a Python pandas dataframe to OpenML, and later get the same dataframe back, without realizing or caring how the data was stored in the meantime. If people want to store the data locally, they can download it in the format they like (e.g. a memory-mapped format like Arrow/Feather for fast reading or TFRecords for people using TensorFlow). Additional code can facilitate such conversions.
- There should be a **standard way to represent specific types of data**, i.e. a fixed schema that can be verified. For instance, all tabular data should be stored in a uniform way. Without it, we would need dataset-specific code for loading, which requires maintenance, and it will be harder to check quality and extract meta-data.
- The format should **allow storing most (processed) machine learning datasets,** including images, video, audio, text, graphs, and multi-tabular data such as object recognition tasks and relational data. Data such as images can be converted to numeric formats (e.g. pixel values) for storage in this format (and usage in machine learning).

**Impact on OpenML (simplicity, maintenance)**

Since OpenML is a community project, we want to keep it as easy as possible to use and maintain:

- We aim to host datasets in an *S3* object store (min.io).
- We prefer a **single internal data format** to reduce maintenance both server-side and client-side.
- We need **machine-readable schemas** (in a specific language) that describe how a certain type of data is formatted. Examples would be a schema for tabular data, a schema for annotated image data, etc. Every dataset should specify the schema it satisfies, and we should be able to **validate** this. We aim to gradually roll out support form different types of data, starting with tabular, and including others only after schemas are defined.
- We need to support batch data now, but ideally the format should allow data appending (streaming) in the future.

When no agreed upon schema exists, we could offer a forum for the community to discuss and agree on a standard schema, in collaboration with other initiatives (e.g. [frictionlessdata](https://frictionlessdata.io/)). For instance, new schemas could be created in a github repo to allow people to do create pull requests. They could be effectively used once they are merged.

**Requirements**

To draw up a shortlist of data formats, we used the following (soft) requirements:

- The format should be stable and **fully maintained** by an active community.
- **Parsers in various programming languages**, including well-maintained and stable libraries.
- **Streaming read/writes**, for easy conversion and memory efficiency.
- **Version control**, some way to **see differences** between versions.
- Ideally, there is a way to **detect bitflip errors** during storage or transmission.
- Ideally, **fast read/writes** and **efficient storage**.
- Ideally, there should be support for storing **sparse data**.
- Support for storing binary blobs and vectors of different lengths.
- If possible, support for multiple 'resources' (e.g. collections of files or multiple relational tables).
- Potentially, store some meta-data inside the file.

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

- Used in industry a lot, active developer community. Good community of practice.
- Well-supported and maintained.
- Has parsers in different languages, but not all Parquet features are supported in every library (see below).
- Built-in compression (columnar storage), very efficient long-term data storage
- Simple structure
- Sparse data

Drawbacks:

- The Python libraries ([Arrow](https://arrow.apache.org/docs/python/parquet.html), [fastparquet](https://fastparquet.readthedocs.io/)) **do not support partial read/writes**. The Java/Go implementations do. Splitting up parquet files into many small files can be cumbersome.
- **No version control, no meta-data storage, no schema enforcement.** There are layers on top (e.g. delta lake) that do support this. Simple file versioning can also be done with S3.
- The different parsers (e.g. [Parquet support inside Arrow](https://arrow.apache.org/docs/python/parquet.html), [fastparquet](https://fastparquet.readthedocs.io/)) implement different parts of the Parquet format and different set of compression algorithms. Hence, parquet files **may not be compatible between parsers** (see [here](https://fastparquet.readthedocs.io/en/latest/#caveats-known-issues) and [here](https://kb.databricks.com/data/wrong-schema-in-files.html).
- Support **limited to single-table storage**. For instance, there doesn't seem to be an apparent way to store an object detection dataset (with images and annotations) as a single parquet file.

[**SQLite**](https://www.sqlite.org/index.html)

Benefits:

- Easy to use and comparably fast to HDF5 in our tests.
- Very good support in all languages. It is [built-in](https://docs.python.org/3.7/library/sqlite3.html) in Python.
- Very flexible access to parts of the data. SQL queries can be used to select any subset of the data.

Drawback:

- It supports **only 2000 columns**, and we have quite a few datasets with more than 2000 features. Hence, storing large tabular data will require mapping data differently, which would add a lot of additional complexity.
- Writing SQL queries **requires knowledge of the internal data structure** (tables, keys,...).

[**HDF5**](https://www.hdfgroup.org/solutions/hdf5/)

Benefits:

- Very good support in all languages. Has well-tested parsers, all using the same C implementation.
- Widely accepted format in the deep learning community to store data and models.
- Widely accepted format in many scientific domains (e.g. astronomy, bioinformatics,...)
- Provides built-in compression. Constructing and loading datasets was reasonably fast.
- Very flexible. Should allow to store any machine learning dataset as a single file.
- Allows easy inclusion of meta-data inside the file, creating a self-contained file.
- Self-descriptive: the structure of the data can be easily read programmatically. For instance, 'dump -H -A 0 mydata.hdf5' will give you a lot of detail on the structure of the dataset.

Drawbacks:

- Complexity. We **cannot make any a priori assumptions about how the data is structured**. We need to define schema and implement code that automatically validates that a dataset follows a specific schema (e.g. using h5dump to see whether it holds a single dataframe that we could load into pandas). We are unaware of any initiatives to define such schema.
- The format has a **very long and detailed specification**. While parsers exist we don't really know whether they are fully compatible with each other.
- Can become corrupt if not carefully used.

**CSV**

Benefits:

- Very good support in all languages.
- Easy to use, requires very little additional tooling
- Text-based, so easy versioning with git LFS. Changes in different versions can be observed with a simple git diff.
- The current standard used in [frictionlessdata](https://frictionlessdata.io/).
- There exist schema to express certain types of data in CSV (see [frictionlessdata](https://frictionlessdata.io/)).

Drawbacks:

- **Not very efficient** for storing floating point numbers
- **Not ideal for very large datasets** (when data does not fit in memory/disk)
- **Many different dialects exist**. We need to decide on a standardized dialect and enforce that only that dialect is used on OpenML ([https://frictionlessdata.io/specs/csv-dialect/](https://frictionlessdata.io/specs/csv-dialect/)). The dialect specified in [RFC4180](https://tools.ietf.org/html/rfc4180), which uses the comma as delimiter and the quotation mark as quote character, is often recommended.



**Overview**

|   | Parquet | HDF5 | SQLite | CSV |
| --- | --- | --- | --- | --- |
| Consistency across <br>different platforms | ? | ✅ | ✅ | ✅ (dialect) |
| Support and documentation | ✅ | ✅ | ✅ | ✅ |
| Read/write speed | ✅ | so-so | ❌ | ❌ |
| Incremental <br>reads/writes | Yes, but not <br>supported by current<br> Python libs | ✅ | ✅ | Yes (but not <br>random access) |
| Supports very large and high-dimensional datasets | ✅ | ✅ | ❌ (limited nr. columns<br> per table) | ✅ Storing tensors<br> requires flattening. |
| Simplicity | ✅ | ❌ (basically full <br>file system) | ✅ (it's a database) | ✅ |
| Metadata support | Only minimal | ✅ | ✅ | ❌ (requires separate <br> metadata file) |
| Maintenance | Apache project, open<br> and quite [active](https://www.slideshare.net/Hadoop_Summit/the-columnar-roadmap-apache-parquet-and-apache-arrow-102997214) | Closed group,<br> but [active](https://www.slideshare.net/HDFEOS/hdf5-roadmap-20192020) community on <br>Jira and conferences | Run by a [company](https://www.sqlite.org/prosupport.html).<br> Uses an email list. | ✅ |
| Available examples of<br> usage in ML | ✅ | ✅ | ❌ | ✅ |
| Flexibility | Only tabular | Very flexible, <br>maybe too flexible | Relational multi-table | Only tabular |
| Versioning/Diff | Only via S3 or delta lake | ❌ | ❌ | ✅ |
| Different length vectors | As blob | ✅ | ❌ ? | ✅ |

**Performance benchmarks**

There exist some prior benchmarks ([here](https://tech.blueyonder.com/efficient-dataframe-storage-with-apache-parquet/) and [here](https://towardsdatascience.com/the-best-format-to-save-pandas-data-414dca023e0d)) on storing dataframes. These only consider single-table datasets. For reading/writing, CSV is clearly slower and Parquet is clearly faster. For storage, Parquet is most efficient but zipped CSV as well. HDF requires a lot more disk space. We also ran our own [benchmark](https://gitlab.com/mitar/benchmark-dataset-formats) to compare the writing performance of those data formats for very large and complex machine learning datasets, but could not find a way to store these in one file in Parquet.

**Version control**

Version control for large datasets is tricky. For text-based formats (CSV), we could use [git LFS](https://git-lfs.github.com/) store the datasets and have automated versioning of datasets. We found it quite easy to export all current OpenML dataset to GitLab: [https://gitlab.com/data/d/openml](https://gitlab.com/data/d/openml).

The binary formats do not allow us to track changes in the data, only to recover the exact versions of the datasets you want (and their metadata). Potentially, extra tools could still be used to export the data to dataframes or text and then compare them. Delta Lake has version history support, but seemingly only for Spark operations done on the datasets.

**We need your help!**
If we have missed any format we should investigate, or misunderstood those we have investigated, or missed some best practice, please tell us.
You are welcome to comment below, or send us an email at openmlhq@googlegroups.com


**Contributors to this blog post:**
Mitar Milutinovic, Prabhant Singh, Joaquin Vanschoren, Pieter Gijsbers, Andreas Mueller, Matthias Feurer, Jan van Rijn, Marcus Weimer, Marcel Wever, Gertjan van den Burg, Nick Poorman
