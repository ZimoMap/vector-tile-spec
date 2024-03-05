# Vector Tile Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in
this document are to be interpreted as described in [RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

## 1. Purpose

This document specifies a memory-efficient encoding format for tiled geographic vector data. It is designed to be used in portable and embedded applications for fast rendering or lookups of feature data.

## 2. File Format

The Vector Tile format uses [Google FlatBuffers](https://flatbuffers.dev/) as a encoding format. FlatBuffers is a cross platform serialization library architected for maximum memory efficiency.

### 2.1. File Extension

The filename extension for Vector Tile files SHOULD be `zvt`. For example, a file might be named `vector.zvt`.

### 2.2. Multipurpose Internet Mail Extensions (MIME)

When serving Vector Tiles the MIME type SHOULD be `application/vnd.zimomap-vector-tile`.

## 3. Projection and Bounds

A Vector Tile represents data based on a square extent within a projection. A Vector Tile SHOULD NOT contain information about its bounds, extent and projection. The file format assumes that the decoder knows the bounds, extent and projection of a Vector Tile before decoding it.

[Web Mercator](https://en.wikipedia.org/wiki/Web_Mercator) is the projection of reference, and [the Google tile scheme](http://www.maptiler.org/google-maps-coordinates-tile-bounds-projection/) is the tile extent convention of reference. Together, they provide a 1-to-1 relationship between a specific geographical area, at a specific level of detail, and a path such as `https://example.com/17/65535/43602.mvt`.

Vector Tiles MAY be used to represent data with any projection and tile extent scheme.

## 4. Internal Structure

This specification describes the structure of data within a Vector Tile. The reader should have an understanding of the [Vector Tile flatbuffers schema document](vector_tile.fbs) and the structures it defines.

### 4.1. Tile

### 4.1. Layers

A Vector Tile consists of a set of named layers. A layer contains geometric features and their metadata.

A Vector Tile SHOULD contain at least one layer. A layer SHOULD contain at least one feature.

A layer MUST contain a `name` field. A Vector Tile MUST NOT contain two or more layers whose `name` values are byte-for-byte identical. Prior to appending a layer to an existing Vector Tile, an encoder MUST check the existing `name` fields in order to prevent duplication.

### 4.2. Features

A feature MUST contain a `geometry` field.

A feature MUST contain a `type` field as described in the Geometry Types section.

A feature MAY contain a `coverage` field as described in the Tile Subgrid section.

A feature MAY contain a `name` field.

A feature MAY contain a `name_zh` field.

A feature MAY contain a `name_en` field.

A feature MAY contain a `pmap_kind` field.

A feature MAY contain a `pmap_min_zoom` field.

### 4.3. Geometry Encoding

Geometry data in a Vector Tile is defined in a screen coordinate system. The upper left corner of the tile (as displayed by default) is the origin of the coordinate system. The X axis is positive to the right, and the Y axis is positive downward. Coordinates within a geometry MUST be integers.

A geometry is encoded as a sequence of 16 bit signed integers in the `geometry` field of a feature. Each integer is either a `CommandInteger` or a `ParameterInteger`. A decoder interprets these as an ordered series of operations to generate the geometry.

A command is always a `MoveTo` command. A `MoveTo` command is an operation that adds a single point to the geometry.

A command sequence is a `CommandInteger` followed by 2 or more `ParameterInteger`. The `CommandInteger` indicates the variant of `MoveTo` command to be added, and the number of times that the command will be excecuted. The `ParameterInteger` encodes the positional parameters for the `MoveTo` command.

#### 4.3.1. Command Integers

A `CommandInteger` indicates a command variant to be added, as a command ID, and the number of times that the command will be executed, as a command count.

A command ID is encoded as an unsigned integer in the least significant 1 bit of the `CommandInteger`, and is either 0 or 1. For `Point`, `MultiPoint`, `LineString` and `MultiLineString` geometries, the command ID MUST be 0. For `Polygon` and `MultiPolygon` geometries, exterior rings MUST have a command ID of 0, while interior rings MUST have a command ID of 1.

A command count is encoded as an unsigned integer in the remaining 14 bits of a `CommandInteger` (the most significant bit is used to represent the sign of the integer), and is in the range `0` through `pow(2, 14) - 1`, inclusive.

A command ID, a command count, and a `CommandInteger` are related by these bitwise operations:

```javascript
CommandInteger = id | (count << 1)
```

```javascript
id = CommandInteger & 0x1
```

```javascript
count = CommandInteger >> 1
```

#### 4.3.2. Parameter Integers

`ParameterInteger`s directly represent the X and Y positions in the screen coordinate system for parsing performance. Each two `ParameterInteger`s encode a single point. For each pair, the first `ParameterInteger` encodes the X coordinate, and the second `ParameterInteger` encodes the Y coordinate. Consecutive pairs of `ParameterIntegers` SHALL NOT repeat the same position as this would create a zero-length line segment. Zigzag encoding is not used with the `ParameterInteger`s.

The number of `ParameterIntegers` that follow a `CommandInteger` is equal to the command count of the `CommandInteger` multiplied by 2. For example, a `CommandInteger` with a with a command count of 3 will be followed by 6 `ParameterIntegers`.

#### 4.3.4. Geometry Types

The `geometry` field is described in each feature by the `type` field which must be a value in the enum `GeomType`. The following geometry types are supported:

* UNKNOWN
* POINT
* LINESTRING
* POLYGON

Geometry collections are not supported.

##### 4.3.4.1. Unknown Geometry Type

The specification purposefully leaves an unknown geometry type as an option. This geometry type encodes experimental geometry types that an encoder MAY choose to implement. Decoders MAY ignore any features of this geometry type.

##### 4.3.4.2. Point Geometry Type

The `POINT` geometry type encodes a point or multipoint geometry. The encoded geometry for a `Point` MUST consist of a single command sequence with a command count greater than 0.

If the command sequence for a `POINT` geometry has a command count of 1, then the geometry MUST be interpreted as a single `Point`; otherwise the geometry MUST be interpreted as a `MultiPoint` geometry, wherein each pair of `ParameterInteger`s encodes a single point.

##### 4.3.4.3. Linestring Geometry Type

The `LINESTRING` geometry type encodes a `LineString` or `MultiLineString` geometry. The encoded geometry for a linestring geometry MUST consist of one or more repetitions of command sequences with a command count greater than 0.

If the command sequence for a `LINESTRING` geometry type includes only a single command sequence then the geometry MUST be interpreted as a single `LineString`; otherwise the geometry MUST be interpreted as a `MultiLineString` geometry, wherein each `CommandInteger` signals the beginning of a new linestring.

##### 4.3.4.4. Polygon Geometry Type

The `POLYGON` geometry type encodes a `Polygon` or `MultiPolygon` geometry, each polygon consisting of exactly one exterior ring that contains zero or more interior rings. The encoded geometry for a polygon consists of one or more repetitions of the following ordered sequence:

1. An `ExteriorRing`
2. 0 or more `InteriorRing`s

Each `ExteriorRing` and `InteriorRing` MUST consist of a single command sequence with a command count greater than 0.

An exterior ring is DEFINED as a linear ring having a positive area as calculated by applying the [surveyor's formula](https://en.wikipedia.org/wiki/Shoelace_formula) to the vertices of the polygon in tile coordinates. In the tile coordinate system (with the Y axis positive down and X axis positive to the right) this makes the exterior ring's winding order appear clockwise.

An interior ring is DEFINED as a linear ring having a negative area as calculated by applying the [surveyor's formula](https://en.wikipedia.org/wiki/Shoelace_formula) to the vertices of the polygon in tile coordinates. In the tile coordinate system (with the Y axis positive down and X axis positive to the right) this makes the interior ring's winding order appear counterclockwise.

If the command sequence for a `POLYGON` geometry type includes only a single exterior ring then the geometry MUST be interpreted as a single `Polygon`; otherwise the geometry MUST be interpreted as a `MultiPolygon` geometry, wherein each exterior ring signals the beginning of a new polygon. If a polygon has interior rings they MUST be encoded directly after the exterior ring of the polygon to which they belong.

Linear rings MUST be geometric objects that have no anomalous geometric points, such as self-intersection or self-tangency. A linear ring SHOULD NOT have an area calculated by the surveyor's formula equal to zero, as this would signify a ring with anomalous geometric points.

Polygon geometries MUST NOT have any interior rings that intersect and interior rings MUST be enclosed by the exterior ring.

#### 4.3.5. Example Geometry Encodings

##### 4.3.5.1. Example Point

An example encoding of a point located at:

* (25, 17)

This would require a single command:

* Command Sequence 1
  * MoveTo(25, 17)

```
Encoded as: [ 2 25 17 ]
              |  |  `> Decoded: 17
              |  `> Decoded: 25
              | ===== MoveTo(25, 17) == create point (25, 17)
              `> [0000001 0] = command id 0 (unused), command count 1
```

##### 4.3.5.2. Example Multi Point

An example encoding of two points located at:

* (5, 7)
* (3, 2)

This would require two commands:

* Command Sequence 1
  * MoveTo(5, 7)
  * MoveTo(3, 2)

```
Encoded as: [ 4 5 7 3 2 ]
              | | | | `> Decoded: 2
              | | | `> Decoded: 3
              | | | === MoveTo(3, 2) == create point (3, 2)
              | | `> Decoded: 7
              | `> Decoded: 5
              | === MoveTo(5, 7) == create point (5,7)
              `> [0000010 0] = command id 0 (unused), command count 2
```

##### 4.3.5.3. Example Linestring

An example encoding of a line with the points:

* (2, 2)
* (2, 10)
* (10, 10)

This would require three commands:

* Command Sequence 1
  * MoveTo(2, 2)
  * MoveTo(2, 10)
  * MoveTo(10, 10)

```
Encoded as: [ 6 2 2 2 10 10 10 ]
              |          ==== MoveTo(10, 10) == Line to Point (10, 10)
              |     ==== MoveTo(2, 10) == Line to Point (2, 10)
              | === MoveTo(2, 2)
              `> [0000011 0] = command id 0 (unused), command count 3
```

##### 4.3.5.4. Example Multi Linestring

An example encoding of two lines with the points:

* Line 1:
  * (2, 2)
  * (2, 10)
  * (10, 10)
* Line 2:
  * (1, 1)
  * (3, 5)

This would require the following commands:

* Command Sequence 1
  * MoveTo(2, 2)
  * MoveTo(2, 10)
  * MoveTo(10, 10)
* Command Sequence 2
  * MoveTo(1, 1)
  * MoveTo(3, 5)

```
Encoded as: [ 6 2 2 2 10 10 10 4 1 1 3 5 ]
              |                |     === MoveTo(3, 5) == Line to Point (3, 5)
              |                | === MoveTo(1, 1)
              |                `> [0000010 0] = command id 0 (unused), command count 2
              |          ===== MoveTo(10, 10) == Line to Point (10, 10)
              |     ==== MoveTo(2, 10) == Line to Point (2, 10)
              | === MoveTo(2, 2)
              `> [0000011 0] = command id 0 (unused), command count 3
```

##### 4.3.5.5. Example Polygon

##### Example Polygon

An example encoding of a polygon with an exterior ring having the points:

* (3, 6)
* (8, 12)
* (20, 34)
* (3, 6) *Path Closing as Last Point*

This would require the following commands:

* Command Sequence 1
  * MoveTo(3, 6)
  * MoveTo(8, 12)
  * MoveTo(20, 34)

```
Encoded as: [ 6 3 6 8 12 20 34 ]
              |          ===== MoveTo(20, 34) == Line to Point (20, 34)
              |     ==== MoveTo(8, 12) == Line to Point (8, 12)
              | === MoveTo(3, 6)
              `> [0000011 0] = command id 0 (exterior ring), command count 3
```

##### 4.3.5.6. Example Multi Polygon

An example of a more complex encoding of two polygons, one with a hole. The position of the points for the polygons are shown below. The winding order of the polygons is VERY important in this example as it signifies the difference between interior rings and a new polygon.

* Polygon 1:
  * Exterior Ring:
    * (0, 0)
    * (10, 0)
    * (10, 10)
    * (0, 10)
    * (0, 0) *Path Closing as Last Point*
* Polygon 2:
  * Exterior Ring:
    * (11, 11)
    * (20, 11)
    * (20, 20)
    * (11, 20)
    * (11, 11) *Path Closing as Last Point*
  * Interior Ring:
    * (13, 13)
    * (13, 17)
    * (17, 17)
    * (17, 13)
    * (13, 13) *Path Closing as Last Point*

This would require the following commands:

* Command Sequence 1 (Exterior Ring)
  * MoveTo(0, 0)
  * MoveTo(10, 0)
  * MoveTo(10, 10)
  * MoveTo(0, 10)
* Command Sequence 2 (Exterior Ring)
  * MoveTo(11, 11)
  * MoveTo(20, 11)
  * MoveTo(20, 20)
  * MoveTo(11, 20)
* Command Sequence 3 (Interior Ring)
  * MoveTo(13, 13)
  * MoveTo(13, 17)
  * MoveTo(17, 17)
  * MoveTo(17, 13)

```
Encoded as: 
[ 8 0 0 10 0 10 10 0 10 8 11 11 20 11 20 20 11 20 9 13 13 13 17 17 17 17 13 ]
  |                     |                         |
  |                     |                         |
  |                     |                         |
  |                     |                         `> [0000100 1] = command id 1 (interior ring), command count 4
  |                     `> [0000100 0] = command id 0 (exterior ring), command count 4
  `> [0000100 0] = command id 0 (exterior ring), command count 4
```

### 4.4. Feature Attributes

Feature attributes are encoded as pairs of integers in the `tag` field of a feature. The first integer in each pair represents the zero-based index of the key in the `keys` set of the `layer` to which the feature belongs. The second integer in each pair represents the zero-based index of the value in the `values` set of the `layer` to which the feature belongs. Every key index MUST be unique within that feature such that no other attribute pair within that feature has the same key index. A feature MUST have an even number of `tag` fields. A feature `tag` field MUST NOT contain a key index or value index greater than or equal to the number of elements in the layer's `keys` or `values` set, respectively.

### 4.5. Example

For example, a GeoJSON feature like:

```json
{
    "type": "FeatureCollection",
    "features": [
        {
            "geometry": {
                "type": "Point",
                "coordinates": [
                    -8247861.1000836585,
                    4970241.327215323
                ]
            },
            "type": "Feature",
            "properties": {
                "hello": "world",
                "h": "world",
                "count": 1.23
            }
        },
        {
            "geometry": {
                "type": "Point",
                "coordinates": [
                    -8247861.1000836585,
                    4970241.327215323
                ]
            },
            "type": "Feature",
            "properties": {
                "hello": "again",
                "count": 2
            }
        }
    ]
}
```

Could be structured like:

```js
layers {
  version: 2
  name: "points"
  features: {
    id: 1
    tags: 0
    tags: 0
    tags: 1
    tags: 0
    tags: 2
    tags: 1
    type: Point
    geometry: 9
    geometry: 2410
    geometry: 3080
  }
  features {
    id: 2
    tags: 0
    tags: 2
    tags: 2
    tags: 3
    type: Point
    geometry: 9
    geometry: 2410
    geometry: 3080
  }
  keys: "hello"
  keys: "h"
  keys: "count"
  values: {
    string_value: "world"
  }
  values: {
    double_value: 1.23
  }
  values: {
    string_value: "again"
  }
  values: {
    int_value: 2
  }
  extent: 4096
}
```

Keep in mind the exact values for the geometry would differ based on the projection and extent of the tile.
