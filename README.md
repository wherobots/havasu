# Havasu Table Format Specification

Havasu is a spatial table format built on top of Apache Iceberg. Havasu
supports ACID transactions, schema evolution, and time travel on spatial
data including geometries and rasters. Havasu stores spatial data in
Apache Parquet format on cloud storage, which is very cost effective and
highly decoupled with computation.

This repository contains the Havasu specification, which is a formal
description of the Havasu table format. The specification is written in
Markdown and can be found in the [spec.md](spec.md) file.

A reader and writer implementation of Havasu table format specification
is available in [WherobotsDB](https://docs.wherobots.com/latest/tutorials/sedonadb/introduction/).
The detailed usage of WherobotsDB Havasu operations can be found on
[Wherobots Havasu References](https://docs.wherobots.com/latest/references/havasu/introduction/).
