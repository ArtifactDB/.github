# Welcome to ArtifactDB!

## About

**ArtifactDB** is a cloud-based data storage solution with versioning, searchable metadata, schema-based validation, and access control.
It was originally developed at [Genentech](https://github.com/genentech) to store genomics datasets and their associated analysis results.
However, the underlying concepts and infrastructure can be used for anything;
if you've got files, and you've got metadata, and you want to share them with others, you can stick it inside an **ArtifactDB**.

**Features:**

- _Cloud-based data store._
  Users can store and access large quantities of data without worrying about provisioning or bandwidth.
  We use [Amazon S3](https://aws.amazon.com/s3/) but the system works with any S3-compatible storage service.
- _Mandatory metadata for each data file._
  Uploaders are required to provide metadata describing the contents of file.
  Metadata quality standards are enforced by validation against [JSON schemas](https://json-schema.org).
- _Project-level versioning with immutability._
  Related files are organized into projects that may have multiple versions.
  Once a given version is uploaded, its associated files and metadata are immutable to ensure reproducibility in downstream pipelines.
- _Project-level access control._
  Project owners can specify the users (or groups of users) who are allowed to see the project.
  Owners can also specify the users who are allowed to create new versions.
- _Searchable metadata._
  We use [Amazon OpenSearch](https://aws.amazon.com/opensearch-service) to index all of our metadata documents,
  providing powerful search functionality that leverages the quality of the supplied metadata.
- _Convenient programmatic interfaces._
  All functionality is wrapped in a REST interface for easy consumption by client applications.

## Try it out!

Let's play around with a test **ArtifactDB** instance to see what it can do.
Specifically, we'll be using the **calcite** demonstration API, which stores common bioinformatics data structures
(such as `SummarizedExperiment`s and `GenomicRanges`) in a language-agnostic format with [ExperimentHub](https://bioconductor.org/packages/ExperimentHub)-like metadata standards.
Note that the demonstration API's functionality is limited because it runs off a bunch of free-tier cloud services,
but it should be good enough to help you decide if the **ArtifactDB** framework is right for you.

### R

Pull the pre-built [Docker image](https://github.com/ArtifactDB/calcite-docker/pkgs/container/calcite-docker%2Fbuilder) containing all the required packages.
Now we can retrieve data from **calcite** storage:

```r
library(calcite)
obj <- fetchObject("test:my_first_sce@v1")
obj
## class: SingleCellExperiment
## dim: 20006 3005
## metadata(1): .internal
## assays(1): counts
## rownames(20006): Tspan12 Tshz1 ... mt-Rnr1 mt-Nd4l
## rowData names(1): featureType
## colnames(3005): 1772071015_C02 1772071017_G12 ... 1772066098_A12
##   1772058148_F03
## colData names(10): tissue group # ... level1class level2class
## reducedDimNames(0):
## mainExpName: endogenous
## altExpNames(2): ERCC repeat
```

We can extract the metadata from the object:

```r
str(objectAnnotation(obj))
## List of 8
##  $ title       : chr "Zeisel mouse brain scRNA-seq"
##  $ description : chr "scRNA-seq data for 3000 cells in the mouse brain, generated by Zeisel et al. (2015) using the Fluidigm C1 technology."
##  $ maintainers :List of 1
##   ..$ :List of 2
##   .. ..$ name : chr "Aaron Lun"
##   .. ..$ email: chr "infinite.monkeys.with.keyboards@gmail.com"
##  $ species     : int 10090
##  $ genome      :List of 1
##   ..$ :List of 2
##   ... etc...
```

We can examine the permissions for the `test` project:

```r
str(zircon::getPermissions("test", url=restURL()))
## List of 5
##  $ scope       : chr "project"
##  $ read_access : chr "public"
##  $ write_access: chr "owners"
##  $ owners      : chr "ArtifactDB-bot"
##  $ viewers     : chr(0) 
```

Uploading a new version is straightforward, provided you're on the list of `owners`:

```r
tmp <- tempfile()
dir.create(tmp)
saveObject(obj, tmp, "my_first_sce")
uploadDirectory(tmp, "test", "v2")
```

Check out the [**calcite** R package](https://github.com/ArtifactDB/calcite-R) for more usage examples.

### Python

???????????? **Under construction** ????????????

### Javascript

The [Javascript client](https://github.com/ArtifactDB/artifactdb.js) provides useful functions for fetching files and metadata from an ArtifactDB API.
For example, we can extract the count matrix of the `SingleCellExperiment`:

```js
import * as adb from "artifactdb";
let calcite_url = "https://calcite.aaron-lun.workers.dev";

// Get the metadata for the SingleCellExperiment:
let meta = await adb.getFileMetadata(calcite_url, "test:my_first_sce@v1");

// Find the count matrix:
let assay = adb.extractByNameOrIndex(meta.summarized_experiment.assays, "counts");
let assay_id = adb.packId("test", assay.resource.path, "v1");

// Download the count matrix as an ArrayBuffer.
let counts = await adb.getFile(calcite_url, assay_id);
```

Parsing of the downloaded file is left to the application.
Here's an example using [**scran.js**](https://github.com/jkanche/scran.js):

```js
import * as scran from "scran.js";
await scran.initialize();
scran.writeFile("counts.h5", new Uint8Array(counts));
let loaded = scran.initializeSparseMatrixFromHDF5("counts.h5", "sparse");
let mat = loaded.matrix;
```

Uploading is similarly easy, provided we have the MD5 checksum and contents of each file we want to upload.
Let's create a new version of the `test` project with some modification of the SCE's metadata.

```js
import * as hash from "hash-wasm";

let existing = adb.getProjectMetadata(calcite_url, "test", { version: "v1" });
let cloned = adb.prepareCloneUpload(existing, { stringify: false });
let contents = cloned.metadata;
let links = cloned.links;

// Modifying the authors.
let new_contributor = { name: "Darth Vader", email: "vader@empire.gov" };
contents["my_first_sce/experiment.json"].maintainers.push(new_contributor);

let checksums = {};
for (const [k, v] of Object.entries(contents)) {
    contents[k] = JSON.stringify(v);
    checksums[k] = await hash.md5(contents[k]);
}

// Initializing the upload, creating links where possible.
await adb.uploadProject(
    calcite_url, 
    "test-public", 
    "my_test_version", 
    checksums, 
    contents, 
    { initArgs: { dedupLinkPaths: links, expires: 1 } }
);
```

## Set up your own instance

Organisations of any size can set up their own **ArtifactDB** instance to manage their data. 
By specifying the schemas, maintainers of each instance can control the types of files that can be uploaded and the metadata expected for each file.
Developers can easily extend the capabilities of any instance by contributing more schemas to handle new data types.
Finally, end users can painlessly interact with the instance via customized client interfaces for fetching, uploading and searching objects and metadata. 

### Defining the schemas

The choice of schemas is the biggest decision for a prospective **ArtifactDB** maintainer.
We have already created a [large set of JSON schemas](https://github.com/ArtifactDB/BiocObjectSchemas) for handling bioinformatics datasets and results,
which can be easily customized to add more metadata (see [example here](https://github.com/ArtifactDB/ExperimentHub-schemas)) or new data structures.
However, any JSON schema can be used to describe any type of file, provided a few basic conditions are met.

### Configuring the backend

If you just want to dip your foot in the water, you can set up a [**gypsum**](https://github.com/ArtifactDB/gypsum-worker) backend.
This provides a light-weight **ArtifactDB** instance that runs off some free-tier cloud services,
which should be enough to experiment with the **ArtifactDB** concept with minimal investment.
Check out the [**calcite**](https://github.com/ArtifactDB/calcite-worker) API for an example of how to customize **gypsum** to a different set of schemas.

If you want to get serious about **ArtifactDB**, try the full-featured **ArtifactDB** backend based on the Amazon stack.
We're currently in the process of open-sourcing this; details coming soon!

### Customizing the clients

We have already implemented several client packages for interacting with **ArtifactDB** instances in various languages.
These packages can be used directly with a custom instance after performing some configuration (e.g., specifying the REST interface's URL).
However, it is often desirable to create wrapper packages to automate this configuration for a more streamlined end user experience.

In R, the [**zircon**](https://github.com/ArtifactDB/zircon-R) package provides methods to interact with an **ArtifactDB** REST API.
In custom instances, this is typically combined with another set of packages that describes how the files should be saved/loaded into the R session.
For example, the [**calcite**](https://github.com/ArtifactDB/calcite-R) package combines **zircon** with the [**alabaster**](https://github.com/ArtifactDB/alabaster.base) framework to easily load and save Bioconductor objects from the **calcite** API.

## Links

**Schemas:**

- [Bioconductor Object Schemas](https://github.com/ArtifactDB/BiocObjectSchemas), 
  which contain schemas for common Bioconductor classes.
- [ExperimentHub-like Schemas](https://github.com/ArtifactDB/ExperimentHub-schemas), 
  which extend the Bioconductor Object Schemas to include [ExperimentHub](https://bioconductor.org/packages/ExperimentHub)-like metadata.

**Backends:**

- [The **gypsum** worker](https://github.com/ArtifactDB/gypsum-worker),
  which implements a light-weight **ArtifactDB** API based on the Cloudflare stack.
- [The **calcite** worker](https://github.com/ArtifactDB/calcite-worker),
  a fork of the **gypsum** worker that uses a different set of schemas.

**Clients:**

- The [**zircon** R package](https://github.com/ArtifactDB/zircon-R),
  which implements a general-purpose interface to the **ArtifactDB** REST API.
- The [**alabaster** set of R packages](https://github.com/ArtifactDB/alabaster.base),
  which save and load common Bioconductor object in language-agnostic file formats based on the Bioconductor Object Schemas.
- The [**calcite** R package](https://github.com/ArtifactDB/calcite-R),
  which wraps **zircon** and **alabaster** for interacting with the **calcite** REST API. 
- The [**artifactdb** Javascript package](https://github.com/ArtifactDB/artifactdb.js),
  which implements helpful functions for fetching and uploading to/from an **ArtifactDB** API.
