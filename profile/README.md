# Storage for analysis-ready artifacts 

## About

This organization contains various repositories for the **ArtifactDB** project, a storage system for analysis-ready artifacts.
It aims to provide easy access to datasets and analysis results across multiple programming frameworks such as R and Python.
The ArtifactDB system was originally developed at [**@Genentech**](https://github.com/Genentech) to store the outputs of various genomics analysis pipelines (plus the associated metadata);
scientists can then pull these artifacts into their analysis environments for further processing.

## For users 

### R

The [**alabaster** suite](https://github.com/ArtifactDB/alabaster.base) implements functions to save and read Bioconductor objects via language-agnostic file formats.
This is the workhorse of our R-based data serialization pipelines, managing the conversion of various objects into files for long-term storage.

The [**gypsum** R package](https://github.com/ArtifactDB/gypsum-R) implements an interface to the [**gypsum**](#for-developers) REST API.
This handles the upload of files and associated metadata to cloud storage for large-scale distribution.

### Python

The [**dolomite** suite](https://github.com/ArtifactDB/dolomite-base) implements functions to save and read Bioconductor objects via language-agnostic file formats.
This is equivalent to **alabaster** for Python and is based heavily on the classes from the [**BiocPy**](https://github.com/BiocPy) project.

## For developers

The [**gypsum** worker](https://github.com/ArtifactDB/gypsum-worker) implements a REST API for storing and serving artifacts via the Cloudflare stack.
This uses R2 for storage and Workers to handle authenticated uploads via flexible permission schemes.

The [Gobbler](https://github.com/ArtifactDB/gobbler) manages artifacts across users on a shared filesystem such as those used in HPC clusters.
This is effectively an on-premise version of **gypsum** that is simpler and more efficient for local applications. 

The [**takane** library](https://github.com/ArtifactDB/takane) contains language-agnostic specifications for all Bioconductor object types.
These are enforced by validator functions written in C++, which are used by both **alabaster** and **dolomite** to verify compliance.
