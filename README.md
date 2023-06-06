# STAC API - Fields Extension Specification

- [STAC API - Fields Extension Specification](#stac-api---fields-extension-specification)
  - [Overview](#overview)
  - [Include/Exclude Semantics](#includeexclude-semantics)
    - [`null` vs. empty vs. missing](#null-vs-empty-vs-missing)
  - [Examples](#examples)
    - [Default fields](#default-fields)
    - [Explicitly get a valid STAC Item](#explicitly-get-a-valid-stac-item)
    - [Exclude geometry](#exclude-geometry)
    - [Minimal subset](#minimal-subset)
    - [Exclude a nested fiels](#exclude-a-nested-fiels)

## Overview

- **Title:** Fields
- **OpenAPI specification:** [openapi.yaml](openapi.yaml)
- **Conformance Classes:**
  - `STAC API - Item Search` binding: <https://api.stacspec.org/v1.0.0-rc.3/item-search#fields>
  - `STAC API - Features` binding: <https://api.stacspec.org/v1.0.0-rc.3/ogcapi-features#fields>
- **Scope:** STAC API - Features, STAC API - Item Search
- **[Extension Maturity Classification](https://github.com/radiantearth/stac-api-spec/tree/main/README.md#maturity-classification):** Candidate
- **Dependencies:**
  - [STAC API - Item Search](https://github.com/radiantearth/stac-api-spec/tree/v1.0.0-rc.3/item-search)
  - [STAC API - Features](https://github.com/radiantearth/stac-api-spec/tree/v1.0.0-rc.3/ogcapi-features)
- **Owner**: none

By default, STAC API endpoints that return Item objects return every field of those Items. However,
Item objects can have hundreds of fields, or large
geometries, and even smaller Item objects can add up when large numbers of them are in results. Frequently, not all
fields in an Item are used, so this
specification provides a mechanism for clients to request that servers to explicitly include or exclude certain fields.

This behavior may be bound to either or both of 
[STAC API - Item Search](https://github.com/radiantearth/stac-api-spec/tree/v1.0.0-rc.3/item-search) (`/search` endpoint) or
[STAC API - Features](https://github.com/radiantearth/stac-api-spec/tree/v1.0.0-rc.3/ogcapi-features)
(`/collections/{collectionId}/items` endpoint) by advertising the relevant conformance class. 

When used in a POST request with `Content-Type: application/json`, this adds an attribute `fields` with 
an object value to the core JSON search request body. The `fields` object contains two attributes with string array 
values, `include` and `exclude`.

When used with GET, the semantics are the same, except the syntax is a single parameter `fields` with 
a comma-separated list of field names, where `exclude` values are those prefixed by a `-` and `include` values are 
those with no prefix, e.g., `-geometry`, or `id,-geometry,properties`.

It is recommended that implementations provide exactly the `include` and `exclude` sets specified by the request, 
but this is not required. These values are only hints to the server as to the desires of the client, and not a 
contract about what the response will be. Implementations are still considered compliant if fields not specified as part of `include` 
are in the response or ones specified as part of `exclude` are.  For example, implementations may choose to always 
include simple string fields like `id` and `type` regardless of the `exclude` specification. However, it is recommended 
that implementations honor excludes for fields with more complex and arbitrarily large values 
(e.g., `geometry`, `assets`).  For example, some Item objects may have a geometry with a simple 5 point polygon, but these 
polygons can be very large when reprojected to EPSG:4326, as in the case of a highly-decimated sinusoidal polygons.
Implementations are also not required to implement semantics for nested values whereby one can include an object, but
exclude fields of that object, e.g., include `properties` but exclude `properties.datetime`.

No error must be returned if a specified field has no value for it in the catalog. For example, if the field 
"properties.eo:cloud_cover" is specified but there is no cloud cover value for an Item, a successful HTTP response
must be returned and the Item entities will not contain that 
field. 

If no `fields` are specified, the response is **must** be a valid
[ItemCollection](https://github.com/radiantearth/stac-spec/tree/v1.0.0-rc.3/itemcollection/README.md). If a client excludes
fields that are required in a STAC Item, the server may return an invalid STAC Item. For example, if `type` 
and `geometry` are excluded, the entity will not even be a valid GeoJSON Feature, or if `bbox` is excluded then the entity 
will not be a valid STAC Item.

Implementations may return fields not specified, e.g., id, but must avoid anything other than a minimal entity 
representation.

This specification does not yet require the implementation of an "-ables" endpoint (like CQL2 does for queryables)
that defines the names of the
fields that can be selected, so implementations must provide this out-of-band. Implementers may choose to require
fields in Item Properties to be prefixed with `properties.` or not, or support use of both the prefixed and non-prefixed
name, e.g., `properties.datetime` or `datetime`.

## Include/Exclude Semantics

1. If `fields` attribute is specified as an empty string (GET requests) or as an
   empty object or an object with both `include` and `exclude` set to either null
   or an empty array (for POST requests), then the recommended behavior is to include
   only fields `type`, `stac_version`, `id`, `geometry`, `bbox`, `links`, `assets`,
   and `properties.datetime`. If `properties.datetime` is null, then it is recommended
   to include `properties.start_datetime` and `properties.end_datetime`.
   These are the default fields to ensure a valid STAC Item is returned by default.
   Implementations may choose to include other properties, e.g., `properties.created`,
   but the number of default properties fields should be kept to a minimum.
2. If only `include` is specified, these fields should be the only fields included.
   Any additional fields provided beyond those in the `include` list should be kept
   to a minimum, as the caller has explicitly stated they do not need them.
3. For POST requests, if only `exclude` is specified, the specified fields
   should not be included, but every other field available for the Item should be
   included.
4. For POST requests, if `exclude` is specified and `include` is null or an
   empty array, then the `exclude` fields should be excluded from the default set.
   This also applies to GET requests when only `exclude` fields are specified.
5. For nested fields (e.g., `properties.datetime`), the most specific path
   should be honored first, and `include` should be preferred over `exclude`. For
   example:
   1. If a field is in `exclude`, and a nested field of that field is in
    `include`, the nested field should be included, but no other nested
    fields in the field should be included.  For example, if `properties` is
    excluded and `properties.datetime` is included, then `datetime`
    should be the only nested field in `properties`.
   2. If a field is in `include`, and a nested field of that field is in `exclude`, the field
    should be included, and the nested field should be excluded.  For example,
    if `properties` is included and `properties.datetime` is excluded, then
    `datetime` should not be in `properties`, but every other nested field should be.
6. If the same field is present in both `include` and `exclude`, it should be included.
7. If a field is not present in `include`, but it is present in `exclude`, it should be excluded.

### `null` vs. empty vs. missing

There is a semantic difference between a missing value (i.e., if `include` is not
in the JSON object), and a `null` or empty value.  The recommended behavior
around missing vs. null and empty values is described in [the section
above](#includeexclude-semantics), but is summarized here for reference. "ALL"
means that all of the Item's fields should be returned; "DEFAULT" means the
default set (i.e. the required fields for a valid STAC Item) should be returned.

| include                   | exclude                   | returned                                                |
| ------------------------- | ------------------------- | ------------------------------------------------------- |
| missing, `null`, or empty | missing, `null`, or empty | DEFAULT                                                 |
| ["a", "b"]                | missing, `null`, or empty | ["a", "b"]                                              |
| missing                   | ["a", "b"]                | ALL except for ["a", "b"]                               |
| `null` or empty           | ["a", "b"]                | DEFAULT except for ["a", "b"]                           |
| ["a", "b"]                | ["a"]                     | ["a", "b"]                                              |
| ["a"]                     | ["a", "b"]                | ["a"]                                                   |
| ["a.b"]                   | ["a"]                     | `a.b` should be the only field in `a`                   |
| ["a"]                     | ["a.b"]                   | `a` should be included, but it should not include `a.b` |

In this example, `include` is missing:

```json
{
  "fields": {
    "exclude": ["geometry"]
  }
}
```

In these two examples, `include` is `null` and empty, respectively:

```json
{
  "fields": {
    "include": null,
    "exclude": ["geometry"]
  }
}
```

```json
{
  "fields": {
    "include": [],
    "exclude": ["geometry"]
  }
}
```

The special case of both `include` and `exclude` missing is possible both in
JSON and text. In JSON:

```json
{
  "fields": {}
}
```

or

```json
{
  "fields": null
}
```

In text:

```text
?fields=
```

It is not possible to differentiate between missing, null, and empty for the
text representation, so implementations should assume the value is empty, NOT
missing. For example, this is a case of `include` being empty (NOT missing):

```text
?fields=-geometry
```

## Examples

### Default fields

Return the default fields. This should return valid STAC Item entities.

Query Parameters

```text
?fields=
```

JSON

```json
{
  "fields": {
  }
}
```

### Explicitly get a valid STAC Item

Because implementations may choose to always include other fields (e.g.,
extension-specific fields such as
[sar](https://github.com/stac-extensions/sar)), this could has the same effect
as an empty object for `fields`.

Query Parameters

```text
?fields=id,type,geometry,bbox,properties.datetime,links,assets,stac_version
```

JSON

```json
{
  "fields": {
    "include": [
      "id",
      "type",
      "geometry",
      "bbox",
      "properties.datetime",
      "links",
      "assets",
      "stac_version",
    ]
  }
}
```

### Exclude geometry

Exclude `geometry` from the full item.  This will return an entity that is not a valid GeoJSON Feature or a valid STAC Item.

Query Parameters

```text
?fields=-geometry
```

JSON

```json
{
  "fields": {
    "exclude": [
      "geometry"
    ]
  }
}
```

### Minimal subset

Return the `id`, `type`, `geometry`, and the Properties field `eo:cloud_cover`.
This is not guaranteed not return a valid STAC Item, since not all required Item
fields are included, but an implementor may choose to return a valid STAC
item anyways.

Query Parameters

```text
?fields=id,type,geometry,properties.eo:cloud_cover
```

JSON

```json
{
  "fields": {
    "include": [
      "id",
      "type",
      "geometry",
      "properties.eo:cloud_cover"
    ]
  }
}
```

### Exclude a nested fiels

To include `id` and all the properties fields, except for the `foo` field.

Query Parameters

```text
?fields=id,properties,-properties.foo
```

also valid:

```text
?fields=+id,+properties,-properties.foo
```

JSON

```json
{
  "fields": {
    "include": [
      "id",
      "properties"
    ],
    "exclude": [    
      "properties.foo"
    ]
  }
}
```
