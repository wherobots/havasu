# Havasu Table Spec

This is a specification for the Havasu table format that extends [Iceberg table spec](https://iceberg.apache.org/spec/) to support managing large spatial datasets as tables.

## Rationale

Havasu spec extends both version 1 and version 2 of Iceberg spec. This spec is not applicable to any future versions of Iceberg spec.

Havasu spec only defines data type mappings for Parquet. It does not define data type mappings for other formats such as Avro or ORC. We assume that the data files in Havasu tables are always stored in Parquet format.

## Version

This is version 0.1.0 of the Havasu specification. The version of Havasu specification is stored in the table metadata of Havasu tables. Please refer to [Table Metadata](#table-metadata) for more details.

## Overview

Havasu extends Iceberg spec in the following ways:

* **Primitive spatial data types**: Havasu spec extends Iceberg spec to support spatial data types.
* **Spatial transformation**: Havasu spec extends Iceberg spec to support spatial transformation functions. These functions are used to partition or cluster spatial data.
* **Spatial statistics**: Havasu spec extends the Iceberg manifest files to support spatial statistics.
* **Parquet data type mappings for spatial types**: Havasu spec defines how to represent spatial types in Parquet files.

All other aspects of Iceberg spec are unchanged. For example, Havasu spec does not change the fundamental organization of table files.

## Specification

### Terms

* **Geometry**: A geometry is a spatial data type that represents a geometric shape. A geometry can be a point, a line, a polygon, or a collection of points, lines, or polygons. The geometry type conforms to the [OGC Simple Features for SQL specification](https://www.ogc.org/standard/sfs/).
* **Raster**: A raster is a spatial data type that represents a grid of cells, as well as the geo-referencing information for aligning the cells to locations on the planet. Each cell in a raster contains a value. A raster may contain multiple grids (bands), all the grids should have the same cell data type and shape.
* **MBR**: A minimum bounding rectangle (MBR) is the smallest rectangle that contains a geometry or a raster. An MBR is defined by two points: the lower left corner and the upper right corner. We may use the term "bounding box" or "envelope" interchangeably with MBR.
* **CRS**: A coordinate reference system (CRS) is a coordinate-based local, regional or global system used to locate geographical entities. A CRS is usually defined by a coordinate system, a datum, and a set of transformations between different coordinate systems. A CRS should conform to ISO 19111:2019 and could be represented by a [Well-known Text (WKT)](https://www.ogc.org/standard/wkt-crs/) string.

### Primitive spatial data types

Havasu spec defines the following primitive spatial data types:

| Primitive type     | Description               | Requirements   |
|--------------------|---------------------------|--------------------------------------------------|
| **`geometry`**     | geometry shape            | Conforming to the [OGC Simple Features for SQL specification](https://www.ogc.org/standard/sfs/) |
| **`raster`**       | raster as matrices of pixels with geo-referencing metadata | The data model of rasters were introduced in section [Raster data model](#raster-data-model) |

`geometry` fields in Havasu data files can have various physical types and encodings. The physical type and encoding of a `geometry` field is defined by the `havasu.geometry-encoding` property in the JSON representation of the field. Please refer to [JSON serialization](#json-serialization) for the JSON representation of Havasu schemas.

### Data models

#### Geometry data model

Geometry value consists of a optional spatial reference ID (abbreviated as SRID) and a geometry shape. The SRID is a 32-bit integer that identifies the coordinate system that the geometry shape is using. The interpretation of SRID is implementation dependent. For example, the SRID could be an EPSG code, or a code defined by a proprietary coordinate system.

The geometry shape is one of the following types defined by [OGC Simple Features for SQL specification](https://www.ogc.org/standard/sfs/):

| Geometry type | Description | Example in WKT |
|---------------|-------------|---------|
| `POINT` | A point | `POINT(1 1)` |
| `LINESTRING` | A line | `LINESTRING(1 1, 2 2)` |
| `POLYGON` | A polygon | `POLYGON((1 1, 2 1, 2 2, 1 2, 1 1))` |
| `MULTIPOINT` | A collection of points | `MULTIPOINT((1 1), (2 2))` |
| `MULTILINESTRING` | A collection of lines | `MULTILINESTRING((1 1, 2 2), (3 3, 4 4))` |
| `MULTIPOLYGON` | A collection of polygons | `MULTIPOLYGON(((1 1, 2 1, 2 2, 1 2, 1 1)), ((3 3, 4 3, 4 4, 3 4, 3 3)))` |
| `GEOMETRYCOLLECTION` | A collection of geometries | `GEOMETRYCOLLECTION(POINT(1 1), LINESTRING(1 1, 2 2))` |

The geometry shape can be encoded in various formats in data files. Please refer to [Parquet data type mappings](#parquet-data-type-mappings) for the encoding of geometry values in Parquet files.

#### Raster data model

The raster data model of Havasu specification is largely inspired by the raster data type of PostGIS. A raster value is composed by the following components:

| Field | Type | Description |
|-------|------|-------------|
| `width` | `int` | The width of the raster in pixels |
| `height` | `int` | The height of the raster in pixels |
| `crs` | - | The coordinate reference system of the raster |
| `scale_x` | `double` | The scale factor of the raster in X direction |
| `scale_y` | `double` | The scale factor of the raster in Y direction |
| `skew_x` | `double` | The skew factor of the raster in X direction |
| `skew_y` | `double` | The skew factor of the raster in Y direction |
| `upperleft_x` | `double` | The X coordinate of the upper left corner of the raster |
| `upperleft_y` | `double` | The Y coordinate of the upper left corner of the raster |
| `bands` | `array<struct>` | The bands of the raster |

A raster is one or more grids of cells. All the grids should have `width` rows and `height` columns. The grid cells are represented by the `band` field. The grids are geo-referenced using an affine transformation that maps the grid coordinates to world coordinates. The coordinate reference system (CRS) of the world coordinates is specified by the `crs` field. The CRS will be serialized as a WKT string when stored in data files.

The geo-referencing information is represented by the parameters of an affine transformation (`upperleft_x`, `upperleft_y`, `scale_x`, `scale_y`, `skew_x`, `skew_y`). Havasu specification only supports affine transformation as geo-referencing transformation, other transformations such as polynomial transformation are not supported.

The grid coordinates of a raster is always anchored at the center of grid cells. The translation factor of the affine transformation `upperleft_x` and `upperleft_y` also designates the world coordinate of the center of the upper left grid cell. Havasu specification only requires that the stored raster values in data files should follow this convention, but it does not require that the applications should expose raster values to the users in this convention. For example, SedonaDB exposes raster functions that follows the convention that the grid coordinates of a raster is anchored at the upper left corner of grid cells, which is consistent with GDAL and PostGIS.

Havasu supports persisting raster band values in two different ways:

* **in-db**: The band values are stored in the same data file as the geo-referencing information. The band values are stored in the `bands` field of the raster value.
* **out-db**: The band values are stored in files external to Havasu tables. The raster value stored in Havasu data file contains the geo-referencing information and URI of external raster files. The URI of external raster files are stored in the `bands` field of the raster value.

Havasu specification allows mixture of in-db and out-db bands in the same raster, and also allows different out-db bands of the same raster to be stored in different external files. However, the implementation of Havasu specification may not support all these cases.

The components of in-db band value is shown in the following table:

| Field | Type | Description |
|-------|------|-------------|
| `pixel_type` | `enum` | The type of pixel values |
| `no_data_value` | `pixel_value_type` | The value of no-data cells |
| `data` | `array<pixel_value_type>` | The value of cells |

The `pixel_type` field is an enumeration that specifies the type of pixel values. The pixel types are defined as follows. `pixel_type` is stored as an integer in data files.

| Pixel type | Description |
|------------|-------------|
| `PT_8BSI` | Signed 8-bit integer |
| `PT_8BUI` | Unsigned 8-bit integer |
| `PT_16BSI` | Signed 16-bit integer |
| `PT_16BUI` | Unsigned 16-bit integer |
| `PT_32BSI` | Signed 32-bit integer |
| `PT_32BUI` | Unsigned 32-bit integer |
| `PT_32BF` | 32-bit floating point |
| `PT_64BF` | 64-bit floating point |

The components of out-db band value is shown in the following table:

| Field | Type | Description |
|-------|------|-------------|
| `pixel_type` | `enum` | The type of pixel values |
| `no_data_value` | `binary` | The value of no-data cells |
| `band_number` | `int` | The 0-based index of the band in the external raster file |
| `uri` | `string` | The URI of the external raster file |

Out-db band refers to one specific band indexed by `band_number` in the raster file referenced by `uri`. The raster file referenced by `uri` should be a valid raster file conforming to the following requirements:

* The raster should have a band indexed by `band_number`.
* The pixel type of the band should be the same as `pixel_type`.
* The CRS of the raster file should be equivalent with the CRS of the raster value.
* The affine transformation of the raster file should be aligned with the affine transformation of the raster value: the `scale_x`, `scale_y`, `skew_x` and `skew_y` parameters of both affine transformations should be the same.

The format of raster file is implementation dependent. For example, the raster file could be a Cloud Optimized GeoTIFF (COG) file.

Havasu specification defines how to stores raster values in parquet data files. The definition of parquet data type for raster is almost identical to the raster data model specified here. Please refer to [Parquet data type mappings](#parquet-data-type-mappings) for the encoding of raster values in Parquet files.

### Table Metadata

Havasu extends the table metadata by adding the following fields:

| v1 | v2 | Field | Type | Description |
|----|----|-------|------|-------------|
| *optional* | *optional* | `havasu.format-version` | `string` | The version of Havasu specification that the table conforms to |

If `havasu.format-version` is not present in the table metadata, then the table conforms to Havasu version 0.1.0.

### Manifests

Havasu spec extends the Iceberg manifest files to support spatial statistics. This section describe the changes to Iceberg manifest files.

Havasu inherits the same manifest file schema from Iceberg. The only difference is that the `data_file` struct in Havasu manifest files contain additional fields to store spatial statistics:

| v1 | v2 | Field id, name | Type | Description |
|----|----|----------------|------|-------------|
| *optional* | *optional* | **`1250 geom_lower_bounds`** | `map<1260: int, 1270: binary>` | Lower bounds of geometry or raster values |
| *optional* | *optional* | **`1280 geom_upper_bounds`** | `map<1290: int, 1300: binary>` | Upper bounds of geometry or raster values |

The boundings of geometry and raster values should be derived using the rules described in the next section.

#### Geometry bounds

The bounds of geometry values should be derived using their minimum bounding rectangles (MBRs). The MBR of a geometry value is defined as the smallest rectangle that contains the geometry value. The SRID of geometry values were ignored when computing the MBRs. The following pseudo-code shows how to compute the MBR of a geometry value:

```
min_x = min(x for x, y in geom)
min_y = min(y for x, y in geom)
max_x = max(x for x, y in geom)
max_y = max(y for x, y in geom)
```

The MBRs of all geometry values in a data file should be unioned as a single MBR, which is the MBR of the data file. The MBR of a data file is stored in the `geom_lower_bounds` and `geom_upper_bounds` fields of the manifest file. The lower-left corner of the MBR as a point geometry `POINT (min_x min_y)` should be binary-serialized as the lower bound of the geometry value, and the upper-right corner of the MBR `POINT (max_x max_y)` is serialized as the upper bound of the geometry value. Please refer to [Single spatial value serialization](#single-spatial-value-serialization) for the serialization of geometry values.

#### Raster bounds

Raster bounds are MBRs of rasters in WGS84 coordinate system. They are computed by transforming the envelope of the raster in its native coordinate system to WGS84. The algorithm for transforming the envelopes is implementation dependent. However, there are some requirements for implementing the transformation algorithm:

1. The transformation algorithm should be able to handle the distortion of the envelope caused by the projection of the raster. The transformed envelope on WGS84 should be as inclusive as possible, which means that it covers most of the area covered by the original raster.
2. The transformation algorithm should be able to handle the case that the envelope of the raster covers the poles. The transformed envelope should span [-180, 180] in longitude in such cases.
3. The transformation algorithm should be able to handle the case that the envelope of the raster crosses the anti-meridian. The transformed envelope could span [-180, 180] in longitude, or produces a envelope crossing the anti-meridian described later in such cases.

Raster bounds have a special rule for handling MBRs crossing the anti-meridian. If the X coordinate of lower left corner of the MBR is greater than the X coordinate of the upper right corner of the MBR, then the MBR is considered as crossing the anti-meridian. The area covered by the MBR is made up of two parts: the part on the left side of the anti-meridian and the part on the right side of the anti-meridian. For example, an MBR with `POINT (170, 10)` as lower-left corner and `POINT (-170, 20)` as upper-right corner covers the area from `POINT (170, 10)` to `POINT (180, 20)` on the left side of the anti-meridian, and the area from `POINT (-180, 10)` to `POINT (-170, 20)` on the right side of the anti-meridian. Implementations of Havasu specification should be able to handle MBRs crossing the anti-meridian correctly, otherwise spatial query optimizations will derive incomplete query results.

The MBRs of all raster values in a data file should be unioned as a single MBR, which is the MBR of the data file. The MBR of a data file is stored in the `geom_lower_bounds` and `geom_upper_bounds` fields of the manifest file in the same way as geometry bounds.

#### Scan Planning

Applications can take advantage of the spatial statistics of data files to optimize the query execution plan. For example, if the query predicate is a spatial range query, the application can use the spatial statistics to prune the data files that do not contain any data that satisfies the query predicate. How spatial query optimization is implemented in scan planning is implementation dependent. One possible implementation is to convert the spatial query predicate to an inclusive projection. An inclusive projection is evaluated using the field statistics maintained in manifest files. If a spatial predicate matches at least one row in a data file, then the inclusive query predicate must match that data file. For example, for a spatial range query `ST_Within(geom, Q)`, where `geom` is the geometry field in a Havasu table, `Q` is a constant geometry as the query window, the inclusive projection is `ST_Intersects(MBR[geom], Q)`. `MBR[geom]` is the minimum bounding box of all values of `geom` in a data file. It can be derived from the lower and upper bounds of `geom` stored in `geom_lower_bounds` and `geom_upper_bounds` fields in the manifest file.

Similarly, for spatial range query `ST_Contains(geom, Q)`, the inclusive projection could be `ST_Contains(MBR[geom], Q)`. It is valid to use `ST_Contains` in this case because if geom contains Q, the MBR of all geom values must also contain Q. Although `ST_Intersects(MBR[geom], Q)` is also a valid inclusive projection, `ST_Contains(MBR[geom], Q)` is more appropriate because it is more selective.

For raster range queries, the application may need to transform the query window to WGS84 when converting the spatial predicate to an inclusive projection, since the statistics of raster values are always in WGS84.

## JSON serialization

### Schemas

Havasu extended the serialized representation of Iceberg schemas to support spatial data types.
For the sake of compatibility with Iceberg table format, Havasu encodes spatial data types by adding special properties to the JSON representation of `struct` types.

#### Geometry

For geometry fields in a `struct` type, the `type` of geometry field should be the physical type of the serialized representation of geometry, it could be one of `binary` or `string`. An additional property `havasu.geometry-encoding` was added to the JSON representation of the field to indicate that it is a geometry field. The value of `havasu.geometry-encoding` field is the identifier of geometry serialization format.

For example, the following JSON schema defines a `struct` type with a geometry field named `geom`. The physical type of `geom` is `binary`, and the geometry stored in data files are encoded in EWKB format:

```json
{
  "type": "struct",
  "fields": [
    {
      "id": 1,
      "name": "geom",
      "required": false,
      "type": "BINARY",
      "havasu.geometry-encoding": "ewkb"
    }
  ]
}
```

Valid values of `havasu.geometry-encoding` are:

| Identifier | Description | Compatible physical types |
|------------|-------------|---------------------------|
| `ewkb` | Extended Well-known binary (EWKB) | `binary` |
| `wkb` | Well-known binary (WKB) | `binary` |
| `wkt` | Well-known text (WKT) | `string` |
| `geojson` | GeoJSON | `string` |

The value of `havasu.geometry-encoding` should be consistent with the encoding of geometry values in data files. For encoding of geometry values in parquet files, please refer to [Parquet data type mappings](#parquet-data-type-mappings).

#### Raster

For raster fields in a `struct` type, the `type` of raster field should be the JSON representation of an empty `struct`. An additional property `havasu.raster-encoding` was added to the JSON representation of the field to indicate that it is a raster field. The value of `havasu.raster-encoding` field is the identifier of raster serialization format. The only valid value of `havasu.raster-encoding` is `v1`.

For example, the following JSON schema defines a `struct` type with a raster field named `rast`:

```json
{
  "type": "struct",
  "fields": [
    {
      "id": 1,
      "name": "rast",
      "required": false,
      "type": {
        "type": "struct",
        "fields": []
      },
      "havasu.raster-encoding": "v1"
    }
  ]
}
```

### Parquet Data Type Mappings

Havasu specification defines how to represent spatial types in Parquet files. This section describes the Parquet data type mappings for spatial types.

#### Geometry

Geometry values are stored in parquet according to the encoding property `havasu.geometry-encoding` of geometry field. The following table shows the Parquet data type mappings for geometry values:

| Encoding | Parquet physical type | Logical type | Description |
|----------|-----------------------|--------------|-------------|
| `ewkb` | `BINARY` | | Extended Well-known binary (EWKB) |
| `wkb` | `BINARY` | | Well-known binary (WKB) |
| `wkt` | `BINARY` | `UTF8` | Well-known text (WKT) |
| `geojson` | `BINARY` | `UTF8` | [GeoJSON (RFC 7946)](https://datatracker.ietf.org/doc/html/rfc7946) |

Extended Well-known binary (EWKB) does not have a formal specification. It is a superset of WKB ([OGC SFA specification](https://www.ogc.org/standard/sfa/) 1.2.1) that supports including the SRID value. Havasu specification requires that the EWKB values stored in Parquet files should be in the format defined hereby.

The binary format of EWKB is as follows:

```
[byteOrder] [wkbType] [SRID] [wkbValue]
```

The `byteOrder` and `wkbValue` portion is identical with the WKB specification. The `wkbType` is extended to support including an optional `SRID` value. If `wkbType & 0x20000000` is non-zero, then the `SRID` portion is present, otherwise the `SRID` portion is not present and the EWKB is binary sequence is also in WKB.

The `SRID` value is a 32-bit integer that identifies the coordinate system that the geometry shape is using. It is encoded in the byte order specified by `byteOrder`.

When geometry column is at the root of the schema, and the geometry encoding is one of `wkb` and `ewkb`, the application can optionally write the GeoParquet metadata to the Parquet files. The GeoParquet metadata is defined by [GeoParquet specification](https://github.com/opengeospatial/geoparquet/blob/v1.0.0/format-specs/geoparquet.md). Havasu data files written in this way are compatible with GeoParquet specification and can be directly consumed by [GeoParquet readers](https://geoparquet.org/#implementations).

#### Raster

The parquet data type for raster values is determined by the encoding property `havasu.raster-encoding` of raster field. The only version of raster encoding defined by Havasu specification is `v1`. Havasu spec may define new parquet data type for raster values in the future.

Raster values are stored in parquet as a group type with the following fields:

| Field | Required or optional |Parquet physical type |  Logical type | Description |
|-------|----------------------|----------------------|---------------|-------------|
| `width` | *required* | `INT32` |  | The width of the raster in pixels |
| `height` | *required* | `INT32` |  | The height of the raster in pixels |
| `num_bands` | *required* | `INT32` |  | The number of bands in the raster |
| `crs_wkt` | *optional* | `BINARY` | `UTF8` | The [Well-known Text (WKT)](https://www.ogc.org/standard/wkt-crs/) representation of coordinate reference system of the raster |
| `geo_reference` | *required* | `group`  | | The geo-referencing information of the raster |
| `band_1` | *optional* | `group` | | The first band of the raster |
| `band_2` | *optional* | `group` | | The second band of the raster |
| `band_3` | *optional* | `group` | | The third band of the raster |
| `band_4` | *optional* | `group` | | The fourth band of the raster |
| `bands` | *optional* | `repeated group` | | The remaining bands of the raster |

The group type of `geo_reference` field is defined as follows:

| Field | Required or optional |Parquet physical type |  Logical type | Description |
|-------|----------------------|----------------------|---------------|-------------|
| `scale_x` | *required* | `DOUBLE` |  | The scale factor of the raster in X direction |
| `scale_y` | *required* | `DOUBLE` |  | The scale factor of the raster in Y direction |
| `skew_x` | *required* | `DOUBLE` |  | The skew factor of the raster in X direction |
| `skew_y` | *required* | `DOUBLE` |  | The skew factor of the raster in Y direction |
| `upperleft_x` | *required* | `DOUBLE` |  | The X coordinate of the upper left corner of the raster |
| `upperleft_y` | *required* | `DOUBLE` |  | The Y coordinate of the upper left corner of the raster |

Bands of rasters are stored in the `band_1`, `band_2`, `band_3`, `band_4` and `bands` fields. If a raster has less than 4 bands, then the extra `band_N` fields and the `bands` field should be null. If a raster has more than 4 bands, then `band_1` to `band_4` fields should be non-null and stores the band values of the first 4 bands, and the remaining bands should be stored in the `bands` field. The purpose of `band_1` to `band_4` fields is to allow applications to access one particular band of a raster without reading all the bands, it only works when the band being accessed is one of the first 4 bands.

If the application only accesses the metadata of the raster, such as `ST_Metadata(rast)` or `ST_GeoReference(rast)`, then the application does not need to read the `band_N` fields as well as the `bands` field. Applying this projection push-down optimization to Havasu tables can significantly improve the performance of metadata-only queries for raster fields.

The group type of each band is defined as follows:

| Field | Required or optional |Parquet physical type |  Logical type | Description |
|-------|----------------------|----------------------|---------------|-------------|
| `pixel_type` | *required* | `INT32` |  | The type of pixel values |
| `no_data` | *optional* | `BINARY` |  | The NODATA value |
| `data` | *optional* | `BINARY` |  | The value of cells |
| `out_db_band_no` | *optional* | `INT32` |  | The 0-based band number of out-db raster file |
| `out_db_url` | *optional* | `BINARY` | `UTF8` | The URI of the out-db raster file  |

The `pixel_type` field is an enumeration that specifies the type of pixel values. The pixel types are defined as follows.

| Pixel type | Value | Description |
|------------|-------|-------------|
| `PT_8BSI` | 3 |Signed 8-bit integer |
| `PT_8BUI` | 4 |Unsigned 8-bit integer |
| `PT_16BSI` | 5 |Signed 16-bit integer |
| `PT_16BUI` | 6 |Unsigned 16-bit integer |
| `PT_32BSI` | 7 |Signed 32-bit integer |
| `PT_32BUI` | 8 |Unsigned 32-bit integer |
| `PT_32BF` | 10 |32-bit floating point |
| `PT_64BF` | 11 |64-bit floating point |

The pixel data were serialized using the little-endian bit pattern of values. The number of bytes for each pixel is determined by the value of `pixel_type`. For `PT_32BF` and `PT_64BF` pixel values, the binary representation of pixel values are IEEE 754 compliant bit representation of the float or double values.

The `no_data` field is a binary value that represents the value of no-data cells. If the band does not have NO DATA value, then the `no_data` field should be null.

In-db and out-db bands share the same group type for band values. Here is the rule for determining whether a band is in-db or out-db:

* If `data` is non-null, then the band is in-db.
* Otherwise, the band is out-db. `out_db_band_no` and `out_db_url` must be non-null in this case.

**In-db band**

The `data` field is a binary value that represents the values of all cells in the band. The values of cells are stored in row-major order. The size of `data` field is `width * height * pixel_size`, where `pixel_size` is the size of a pixel value in bytes.

**Out-db band**

The `out_db_band_no` field corresponds to the `band_number` field in raster data model, and the `out_db_url` field corresponds to the `uri` field in raster data model.

## Single spatial value serialization

### Geometry

#### Binary single value serialization

This serialization scheme is for storing single values as individual binary values in the lower and upper bounds maps of manifest files. Binary serialized geometry values are always point values in practice. Geometry values are serialized as binary values by following the Well-known Binary (WKB) specification defined by [Simple Feature Access – Part 1: Common Architecture](https://www.ogc.org/standard/sfa/).

#### JSON single value serialization

Havasu specification does not define how geometry values are serialized, so it is not possible to specify non-null `initial-value` or `write-default` values for geometry fields in Havasu schema. Initial values and write default values for geometry fields are rarely used in practice, so this limitation should not be a problem in most cases.

### Raster

Havasu specification does not define how raster values are serialized, so it is not possible to specify non-null `initial-value` or `write-default` values for raster fields in Havasu schema. Initial values and write default values for raster fields are rarely used in practice, and there is no need to encode raster values in manifest files since the statistics of raster values are geometry values, so this limitation should not be a problem in most cases.
