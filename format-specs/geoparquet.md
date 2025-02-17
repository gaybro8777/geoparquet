# GeoParquet Specification

## Overview

The [Apache Parquet](https://parquet.apache.org/) provides a standardized open-source columnar storage format. The GeoParquet specification defines how geospatial data should be stored in parquet format, including the representation of geometries and the required additional metadata.

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED",  "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

## Version and schema

This is version 1.0.0-dev of the GeoParquet specification.  See the [JSON Schema](schema.json) to validate metadata for this version.

## Geometry columns

Geometry columns MUST be stored using the `BYTE_ARRAY` parquet type. They MUST be encoded as [WKB](https://en.wikipedia.org/wiki/Well-known_text_representation_of_geometry#Well-known_binary).
See the [encoding](#encoding) section below for more details.

### Nesting

Geometry columns MUST be at the root of the schema. A geometry MUST NOT be a group field or nested in a group. In practice, this means that when writing to GeoParquet from another format, geometries cannot be contained in complex or nested types such as structs, lists, arrays, or map types.

### Repetition

The repetition for all geometry columns MUST be "required" (exactly one) or "optional" (zero or one). A geometry column MUST NOT be repeated. A GeoParquet file MAY have multiple geometry columns with different names, but those geometry columns cannot be repeated.

## Metadata

GeoParquet files include additional metadata at two levels:

1. File metadata indicating things like the version of this specification used
2. Column metadata with additional metadata for each geometry column

A GeoParquet file MUST include a `geo` key in the Parquet metadata (see [`FileMetaData::key_value_metadata`](https://github.com/apache/parquet-format#metadata)).  The value of this key MUST be a JSON-encoded UTF-8 string representing the file and column metadata that validates against the [GeoParquet metadata schema](schema.json). The file and column metadata fields are described below.

## File metadata

|     Field Name     |  Type  |                             Description                              |
| ------------------ | ------ | -------------------------------------------------------------------- |
| version     		 | string | **REQUIRED.** The version identifier for the GeoParquet specification. |
| primary_column     | string | **REQUIRED.** The name of the "primary" geometry column. In cases where a GeoParquet file contains multiple geometry columns, the primary geometry may be used by default in geospatial operations. |
| columns            | object\<string, [Column Metadata](#column-metadata)> | **REQUIRED.** Metadata about geometry columns. Each key is the name of a geometry column in the table. |

At this level, additional implementation-specific fields (e.g. library name) MAY be present, and readers should be robust in ignoring those.

### Column metadata

Each geometry column in the dataset MUST be included in the `columns` field above with the following content, keyed by the column name:

| Field Name     | Type         | Description |
| -------------- | ------------ | ----------- |
| encoding       | string       | **REQUIRED.** Name of the geometry encoding format. Currently only `"WKB"` is supported. |
| geometry_types | \[string]    | **REQUIRED.** The geometry types of all geometries, or an empty array if they are not known. |
| crs            | object\|null | [PROJJSON](https://proj.org/specifications/projjson.html) object representing the Coordinate Reference System (CRS) of the geometry. If the field is not provided, the default CRS is [OGC:CRS84](https://www.opengis.net/def/crs/OGC/1.3/CRS84), which means the data in this column must be stored in longitude, latitude based on the WGS84 datum. |
| orientation    | string       | Winding order of exterior ring of polygons. If present must be `"counterclockwise"`; interior rings are wound in opposite order. If absent, no assertions are made regarding the winding order. |
| edges          | string       | Name of the coordinate system for the edges. Must be one of `"planar"` or `"spherical"`. The default value is `"planar"`. |
| bbox           | \[number]    | Bounding Box of the geometries in the file, formatted according to [RFC 7946, section 5](https://tools.ietf.org/html/rfc7946#section-5). |
| epoch          | number       | Coordinate epoch in case of a dynamic CRS, expressed as a decimal year. |

#### crs

The Coordinate Reference System (CRS) is an optional parameter for each geometry column defined in GeoParquet format.

The CRS MUST be provided in [PROJJSON](https://proj.org/specifications/projjson.html) format, which is a JSON encoding of [WKT2:2019 / ISO-19162:2019](https://docs.opengeospatial.org/is/18-010r7/18-010r7.html), which itself implements the model of [OGC Topic 2: Referencing by coordinates abstract specification / ISO-19111:2019](http://docs.opengeospatial.org/as/18-005r4/18-005r4.html). Apart from the difference of encodings, the semantics are intended to match WKT2:2019, and a CRS in one encoding can generally be represented in the other.

If CRS is not provided, all coordinates in the geometries MUST use longitude, latitude based on the WGS84 datum, and the default value is [OGC:CRS84](https://www.opengis.net/def/crs/OGC/1.3/CRS84) for CRS-aware implementations.

[OGC:CRS84](https://www.opengis.net/def/crs/OGC/1.3/CRS84) is equivalent to the well-known [EPSG:4326](https://epsg.org/crs_4326/WGS-84.html) but changes the axis from latitude-longitude to longitude-latitude.

Due to the large number of CRSes available and the difficulty of implementing all of them, we expect that a number of implementations will start without support for the optional `crs` field. Users are recommended to store their data in longitude, latitude (OGC:CRS84 or not including the `crs` field) for it to work with the widest number of tools. Data that are more appropriately represented in particular projections may use an alternate coordinate reference system. We expect many tools will support alternate CRSes, but encourage users to check to ensure their chosen tool supports their chosen CRS.

See below for additional details about representing or identifying OGC:CRS84.

The value of this key may be explicitly set to `null` to indicate that there is no CRS assigned to this column (CRS is undefined or unknown).

#### epoch

In a dynamic CRS, coordinates of a point on the surface of the Earth may change with time. To be unambiguous, the coordinates must always be qualified with the epoch at which they are valid.

The optional `epoch` field allows to specify this in case the `crs` field defines a a dynamic CRS. The coordinate epoch is expressed as a decimal year (e.g. `2021.47`). Currently, this specification only supports an epoch per column (and not per geometry).

#### encoding

This is the binary format that the geometry is encoded in. The string `"WKB"`, signifying Well Known Binary is the only current option, but future versions of the spec may support alternative encodings. This SHOULD be the ["OpenGIS® Implementation Specification for Geographic information - Simple feature access - Part 1: Common architecture"](https://portal.ogc.org/files/?artifact_id=18241) WKB representation (using codes for 3D geometry types in the \[1001,1007\] range). This encoding is also consistent with the one defined in the ["ISO/IEC 13249-3:2016 (Information technology - Database languages - SQL multimedia and application packages - Part 3: Spatial)"](https://www.iso.org/standard/60343.html) standard.

Note that the current version of the spec only allows for a subset of WKB: 2D or 3D geometries of the standard geometry types (the Point, LineString, Polygon, MultiPoint, MultiLineString, MultiPolygon, and GeometryCollection geometry types). This means that M values or non-linear geometry types are not yet supported.

#### Coordinate axis order

The axis order of the coordinates in WKB stored in a GeoParquet follows the de facto standard for axis order in WKB and is therefore always (x, y) where x is easting or longitude and y is northing or latitude. This ordering explicitly overrides the axis order as specified in the CRS. This follows the precedent of [GeoPackage](https://geopackage.org), see the [note in their spec](https://www.geopackage.org/spec130/#gpb_spec).

#### geometry_types

This field captures the geometry types of the geometries in the column, when known. Accepted geometry types are: `"Point"`, `"LineString"`, `"Polygon"`, `"MultiPoint"`, `"MultiLineString"`, `"MultiPolygon"`, `"GeometryCollection"`.

In addition, the following rules are used:

- In case of 3D geometries, a `" Z"` suffix gets added (e.g. `["Point Z"]`).
- A list of multiple values indicates that multiple geometry types are present (e.g. `["Polygon", "MultiPolygon"]`).
- An empty array explicitly signals that the geometry types are not known.
- The geometry types in the list must be unique (e.g. `["Point", "Point"]` is not valid).

It is expected that this field is strictly correct. For example, if having both polygons and multipolygons, it is not sufficient to specify `["MultiPolygon"]`, but it is expected to specify `["Polygon", "MultiPolygon"]`. Or if having 3D points, it is not sufficient to specify `["Point"]`, but it is expected to list `["Point Z"]`.

#### orientation

This attribute indicates the winding order of polygons. The only available value is `"counterclockwise"`. All vertices of exterior polygon rings MUST be ordered in the counterclockwise direction and all interior rings MUST be ordered in the clockwise direction.

If no value is set, no assertions are made about winding order or consistency of such between exterior and interior rings or between individual geometries within a dataset. Readers are responsible for verifying and if necessary re-ordering vertices as required for their analytical representation.

Writers are encouraged but not required to set `orientation="counterclockwise"` for portability of the data within the broader ecosystem.

It is RECOMMENDED to always set the orientation (to counterclockwise) if `edges` is `"spherical"` (see below).

#### edges

This attribute indicates how to interpret the edges of the geometries: whether the line between two points is a straight cartesian line or the shortest line on the sphere (geodesic line). Available values are:
- `"planar"`: use a flat cartesian coordinate system.
- `"spherical"`: use a spherical coordinate system and radius derived from the spheroid defined by the coordinate reference system.

If no value is set, the default value to assume is `"planar"`.

Note if `edges` is `"spherical"` then it is RECOMMENDED that `orientation` is always ensured to be `"counterclockwise"`. If it is not set, it is not clear how polygons should be interpreted within spherical coordinate systems, which can lead to major analytical errors if interpreted incorrectly. In this case, software will typically interpret the rings of a polygon such that it encloses at most half of the sphere (i.e. the smallest polygon of both ways it could be interpreted). But the specification itself does not make any guarantee about this.

#### bbox

Bounding boxes are used to help define the spatial extent of each geometry column. Implementations of this schema may choose to use those bounding boxes to filter partitions (files) of a partitioned dataset.

The bbox, if specified, MUST be encoded with an array representing the range of values for each dimension in the geometry coordinates. For geometries in a geographic coordinate reference system, longitude and latitude values are listed for the most southwesterly coordinate followed by values for the most northeasterly coordinate. This follows the GeoJSON specification ([RFC 7946, section 5](https://tools.ietf.org/html/rfc7946#section-5)), which also describes how to represent the bbox for a set of geometries that cross the antimeridian.

For non-geographic coordinate reference systems, the items in the bbox are minimum values for each dimension followed by maximum values for each dimension. For example, given geometries that have coordinates with two dimensions, the bbox would have the form `[<xmin>, <ymin>, <xmax>, <ymax>]`. For three dimensions, the bbox would have the form `[<xmin>, <ymin>, <zmin>, <xmax>, <ymax>, <zmax>]`.

The bbox values are in the same coordinate reference system as the geometry.

### Additional information

#### Feature identifiers

If you are using GeoParquet to serialize geospatial data with feature identifiers, it is RECOMMENDED that you create your own [file key/value metadata](https://github.com/apache/parquet-format#metadata) to indicate the column that represents this identifier. As an example, GDAL writes additional metadata using the `gdal:schema` key including information about feature identifiers and other information outside the scope of the GeoParquet specification.

### OGC:CRS84 details

The PROJJSON object for OGC:CRS84 is:

```json
{
    "$schema": "https://proj.org/schemas/v0.5/projjson.schema.json",
    "type": "GeographicCRS",
    "name": "WGS 84 longitude-latitude",
    "datum": {
        "type": "GeodeticReferenceFrame",
        "name": "World Geodetic System 1984",
        "ellipsoid": {
            "name": "WGS 84",
            "semi_major_axis": 6378137,
            "inverse_flattening": 298.257223563
        }
    },
    "coordinate_system": {
        "subtype": "ellipsoidal",
        "axis": [
        {
            "name": "Geodetic longitude",
            "abbreviation": "Lon",
            "direction": "east",
            "unit": "degree"
        },
        {
            "name": "Geodetic latitude",
            "abbreviation": "Lat",
            "direction": "north",
            "unit": "degree"
        }
        ]
    },
    "id": {
        "authority": "OGC",
        "code": "CRS84"
    }
}
```

For implementations that operate entirely with longitude, latitude coordinates and are not CRS-aware or do not have easy access to CRS-aware libraries that can fully parse PROJJSON, it may be possible to infer that coordinates conform to the OGC:CRS84 CRS based on elements of the `crs` field.  For simplicity, Javascript object dot notation is used to refer to nested elements.

The CRS is likely equivalent to OGC:CRS84 for a GeoParquet file if the `id` element is present:

* `id.authority` = `"OGC"` and `id.code` = `"CRS84"`
* `id.authority` = `"EPSG"` and `id.code` = `4326` (due to longitude, latitude ordering in this specification)

It is reasonable for implementations to require that one of the above `id` elements are present and skip further tests to determine if the CRS is functionally equivalent with OGC:CRS84.

Note: EPSG:4326 and OGC:CRS84 are equivalent with respect to this specification because this specification specifically overrides the coordinate axis order in the `crs` to be longitude-latitude.
