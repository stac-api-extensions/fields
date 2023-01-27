# STAC API - Fields Extension Specification

- [STAC API - Fields Extension Specification](#stac-api---fields-extension-specification)
  - [Overview](#overview)
  - [Include/Exclude Semantics](#includeexclude-semantics)
  - [Examples](#examples)

## Overview

- **Title:** Fields
- **OpenAPI specification:** [openapi.yaml](openapi.yaml)
- **Conformance Classes:**
  - `STAC API - Item Search` binding: <https://api.stacspec.org/v1.0.0-rc.1/item-search#fields>
  - `STAC API - Features` binding: <https://api.stacspec.org/v1.0.0-rc.1/ogcapi-features#fields>
- **Scope:** STAC API - Features, STAC API - Item Search
- **[Extension Maturity Classification](https://github.com/radiantearth/stac-api-spec/tree/main/README.md#maturity-classification):** Candidate
- **Dependencies:**
  - [STAC API - Item Search](https://github.com/radiantearth/stac-api-spec/tree/v1.0.0-rc.1/item-search)
  - [STAC API - Features](https://github.com/radiantearth/stac-api-spec/tree/v1.0.0-rc.1/ogcapi-features)
- **Owner**: none

By default, STAC API endpoints that return Item objects return every field of those Items. However,
Item objects can have hundreds of fields, or large
geometries, and even smaller Item objects can add up when large numbers of them are in results. Frequently, not all
fields in an Item are used, so this
specification provides a mechanism for clients to request that servers to explicitly include or exclude certain fields.

This behavior may be bound to either or both of 
[STAC API - Item Search](https://github.com/radiantearth/stac-api-spec/tree/v1.0.0-rc.1/item-search) (`/search` endpoint) or
[STAC API - Features](https://github.com/radiantearth/stac-api-spec/tree/v1.0.0-rc.1/ogcapi-features)
(`/collections/{collectionId}/items` endpoint) by advertising the relevant conformance class. 

When used in a POST request with `Content-Type: application/json`, this adds an attribute `fields` with 
an object value to the core JSON search request body. The `fields` object contains two attributes with string array 
values, `include` and `exclude`.

When used with GET, the semantics are the same, except the syntax is a single parameter `fields` with 
a comma-separated list of attribute names, where `exclude` values are those prefixed by a `-` and `include` values are 
those with no prefix, e.g., `-geometry`, or `id,-geometry,properties`.

It is recommended that implementations provide exactly the `include` and `exclude` sets specified by the request, 
but this is not required. These values are only hints to the server as to the desires of the client, and not a 
contract about what the response will be. Implementations are still considered compliant if fields not specified as part of `include` 
are in the response or ones specified as part of `exclude` are.  For example, implementations may choose to always 
include simple string fields like `id` and `type` regardless of the `exclude` specification. However, it is recommended 
that implementations honor excludes for attributes with more complex and arbitrarily large values 
(e.g., `geometry`, `assets`).  For example, some Item objects may have a geometry with a simple 5 point polygon, but these 
polygons can be very large when reprojected to EPSG:4326, as in the case of a highly-decimated sinusoidal polygons.
Implementations are also not required to implement semantics for nested values whereby one can include an object, but
exclude attributes of that object, e.g., include `properties` but exclude `properties.datetime`.

No error must be returned if a specified field has no value for it in the catalog. For example, if the attribute 
"properties.eo:cloud_cover" is specified but there is no cloud cover value for an Item, a successful HTTP response
must be returned and the Item entities will not contain that 
attribute. 

If no `fields` are specified, the response is **must** be a valid
[ItemCollection](https://github.com/radiantearth/stac-spec/tree/v1.0.0-rc.1/itemcollection/README.md). If a client excludes
attributes that are required in a STAC Item, the server may return an invalid STAC Item. For example, if `type` 
and `geometry` are excluded, the entity will not even be a valid GeoJSON Feature, or if `bbox` is excluded then the entity 
will not be a valid STAC Item.

Implementations may return attributes not specified, e.g., id, but must avoid anything other than a minimal entity 
representation.

This specification does not yet require the implementation of an "-ables" endpoint (like CQL2 does for queryables)
that defines the names of the
fields that can be selected, so implementations must provide this out-of-band. Implementers may choose to require
fields in Item Properties to be prefixed with `properties.` or not, or support use of both the prefixed and non-prefixed
name, e.g., `properties.datetime` or `datetime`.

## Include/Exclude Semantics 

1. If `fields` attribute is specified with an empty object, or with both `include` and `exclude` set to null or an 
empty array, the recommended behavior is as if `include` was set to 
`["id", "type", "geometry", "bbox", "links", "assets", "properties.datetime"]`.  This default is so that the entity 
returned is a valid STAC Item.  Implementations may choose to add other properties, e.g., `created`, but the number 
of default properties attributes should be kept to a minimum.
2. If only `include` is specified, these attributes are added to the default set of attributes (set union operation). 
3. If only `exclude` is specified, these attributes are subtracted from the union of the default set of attributes and 
the `include` attributes (set difference operation).  This will result in an entity that is not a valid Item if any 
of the excluded attributes are in the default set of attributes.
4. If both `include` and `exclude` attributes are specified, semantics are that a field must be included and **not** 
excluded.  E.g., if `properties` is included and `properties.datetime` is excluded, then `datetime` must not appear 
in the attributes of `properties`.

## Examples

Return baseline fields.  This **must** return valid STAC Item entities. 

Query Parameters
```http
?fields=
```

JSON
```json
{
  "fields": {
  }
}
```

This has a similar effect as an empty object for `fields`, but it is up to the discretion of the implementation 

Query Parameters
```http
?fields=id,type,geometry,bbox,properties,links,assets
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
      "properties",
      "links",
      "assets"
    ]
  }
}
```

Exclude `geometry` from the baseline fields.  This **must** return an entity that is not a valid GeoJSON Feature or a valid STAC Item.

Query Parameters
```http
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

To return the `id`, `type`, `geometry`, and the Properties attribute `eo:cloud_cover`.
This **must** return a valid STAC Item, as the includes are added to the default includes.
Explicitly specifying `id`, `type`, and `geometry` has not effect as these are default fields,
but `properties.eo:cloud_cover` is not a default field and thereby should be in the response.

Query Parameters
```http
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

To include `id` and all the properties fields, except for the `foo` field.

Query Parameters
```http
?fields=id,properties,-properties.foo
```

also valid:
```http
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
