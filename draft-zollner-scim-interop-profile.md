---
title: SCIM 2.0 Interoperability Profile
abbrev: SCIM Interop Profile
ipr: trust200902
area: Applications and Real-Time
workgroup: SCIM
keyword:
  - scim
  - provisioning
  - identity

pi: [toc, sortrefs, symrefs]

docname: draft-ietf-scim-interop-profile-latest

author:
  - 
    name: Danny Zollner
    organization: Okta
    email: danny.zollner@okta.com

normative:
  RFC7644:
  RFC7643:
  RFC9865:
  RFC5246:
  
---


--- abstract

This document defines an implementation profile for the System for Cross-domain Identity Management (SCIM) 2.0. The goal of this profile is to increase interoperability between identity providers and service providers by reducing the number of optional features and providing clear guidance on implementing a common subset of the SCIM standard. It deprecates certain features that have proven to be problematic for interoperability or are considered insecure.

--- middle

# Discussion Venues

This note is to be removed before publishing as an RFC.

Source for this draft and an issue tracker can be found at https://github.com/Zollnerd/scim-interop-profile.

# Introduction

The SCIM 2.0 standard [RFC7643] [RFC7644] provides a powerful and flexible framework for automating user provisioning. However, its flexibility, with numerous optional features, attributes, and protocol variations, has led to significant interoperability challenges. Implementers are often faced with a wide array of choices, resulting in bespoke integrations that are costly to build and maintain.

This document specifies a profile for SCIM 2.0 to address these challenges. It provides a baseline of required features and deprecates others to ensure that implementations conforming to this profile can interoperate seamlessly.

The target audience for this profile includes developers of identity providers (IDPs) and service providers (SPs) who wish to build conformant and interoperable SCIM clients and servers.

# Notational Conventions

{::boilerplate bcp14-tagged}

# Scope and Conformance

This profile applies to SCIM 2.0 Service Providers and Clients.

A Service Provider is conformant with this profile if it implements all the "MUST" and "REQUIRED" features defined herein.

A Client is conformant with this profile if it is capable of interacting with a conformant Service Provider.

Implementations claiming conformance to this profile should indicate so in their `ServiceProviderConfig` response.

# Data Model Requirements

## Supported Resource Types

Conformant Service Providers MUST implement the following resource types and their corresponding endpoints:

*  `User` ([RFC7643], Section 4.1)

Service Providers MUST publish an accurate list of schemas and attributes via the `/Schemas` endpoint, matching exactly what is implemented and supported by the service.

Conformant Service Providers MAY implement the following resource types and their corresponding endpoints:

*  `Group` ([RFC7643], Section 4.2)

The following configuration discovery-related resource types and their corresponding endpoints MUST be implemented:

*  `ServiceProviderConfig` ([RFC7643], Section 4)
*  `Schema` ([RFC7643], Section 7)
*  `ResourceType` ([RFC7643], Section 6)

## Attribute Requirements

This section will define the minimal set of attributes that MUST be supported for the `User` and `Group` resources to ensure a baseline level of interoperability.

### User Attributes

To ensure a functional baseline for user provisioning, Service Providers **MUST** support the following attributes for the `User` resource:

* `userName`
* `active`
* `displayName`
* `name.givenName`
* `name.familyName`

The `password` attribute is deprecated and **MUST NOT** be implemented. Service Providers **SHOULD NOT** store user passwords and should rely on other authentication methods, such as federation via SAML or OpenID Connect, to authenticate users.

### Group Attributes

Service Providers MUST support both the `displayName` and `members` attributes. All group resources MUST contain a value for `displayName`. Service Providers MUST allow groups to be created without any members.   

## Case Sensitivity

To ensure predictable and interoperable behavior, Service Providers **MUST** implement case sensitivity consistently across both filtering operations and uniqueness constraint enforcement. For any given attribute, the case sensitivity rules applied during a filter query (e.g., `userName eq "User.A"`) **MUST** be identical to the rules used to detect a uniqueness conflict during a `POST` or `PATCH` operation.

Furthermore, this profile requires adherence to the case sensitivity definitions specified in [RFC7643] for the following common attributes:
*   `userName`: `caseExact` is `false`. Service Providers **MUST** treat `JSmith` and `jsmith` as equivalent.
*   `externalId`: `caseExact` is `true`. Service Providers **MUST** treat `ABC-123` and `abc-123` as distinct values.

## Attribute and Schema Handling

To ensure strict conformance and prevent unintended data loss or corruption, Service Providers **MUST** adhere to a strict handling model for attributes and schema extensions. When a SCIM request (e.g., `POST`, `PUT`, `PATCH`) contains attributes or schema URIs that are not defined in the Service Provider's `ResourceType` or `Schema` definitions, the request **MUST** be rejected.

The Service Provider **MUST** return an HTTP `400 Bad Request` with a `scimType` error of `invalidSyntax` for such requests.

## Canonical Values for Typed Attributes

For multi-valued attributes that include a `type` sub-attribute (e.g., `emails`, `phoneNumbers`, `ims`, `photos`), Clients **MUST** use the canonical `type` values defined in [RFC7643] (e.g., "work", "home", "other"). Service Providers **MUST** treat these canonical values as case-insensitive.

# Protocol and Endpoint Requirements

## Data Format and HTTP Headers

All data exchange MUST use the JSON format as defined in [RFC7643], and all requests and responses containing SCIM data MUST use the `Content-Type` header with the value `application/scim+json` as defined in [RFC7644], Section 8.1. The XML data format is explicitly out of scope for this profile and MUST NOT be used.

To aid in troubleshooting and client identification, SCIM clients MUST include a `User-Agent` header in all HTTP requests. The header's value should be meaningful, for example, identifying the name of the client software.

## Filtering

The `filter` query parameter MUST be supported. Service Providers MUST support the `eq` and `and` operators. To promote interoperability, clients MUST NOT use operators other than `eq` and `and`.

The use of filters in the URI for any HTTP method other than `GET` (e.g., `PATCH /Users?filter=...`) is deprecated and **MUST NOT** be used.

Service Providers MUST support filtering on the following User attributes:

* 'userName'
* 'emails.value'
* 'emails.type'
* 'externalId'

Service Providers MUST support filtering on the following Group attributes:

* 'displayName'
* 'members.value'
* 'externalId'

## Pagination

To ensure the reliable handling of large data sets, service providers MUST implement at least one of the following pagination methods for all list operations:

* Index-based pagination as defined in [RFC7644], Section 3.4.2.4.
* Cursor-based pagination, as defined in [RFC9865].

If the `count` parameter is omitted from a request, Service Providers SHOULD return at least 100 results by default. Service Providers MUST support a client-requested `count` value of at least 250. Service Providers SHOULD enforce a server-specified maximum number of results per page and MUST return fewer results than requested when the client specifies a `count` value that exceeds that limit.

## Updating Resources

Service Providers MUST support the `PATCH` operation ([RFC7644], Section 3.5.2) for resource updates. Clients MUST use the `PATCH` operation for updates and SHALL NOT use the `PUT` operation.

This profile defines a restricted subset of the SCIM 2.0 `PATCH` method to ensure predictable behavior and high interoperability.

### General PATCH Constraints

#### Mandatory Use of the 'path' Attribute
Every operation object within the `Operations` array **MUST** contain a `path` attribute. Clients **MUST NOT** issue "path-less" PATCH operations where the target attribute is implied by the keys within the `value` object. 

Service Providers **MUST** reject PATCH requests containing operations that lack a `path` attribute with an HTTP `400 Bad Request` and a `scimType` error of `invalidSyntax`.

#### Multi-Attribute Updates
When a Client needs to update multiple attributes in a single HTTP request, it **MUST** provide a separate operation entry within the `Operations` array for each unique attribute path.

### Attribute-Specific Requirements

#### Singular Attributes (Simple and Complex)
1.  **Simple Attribute Operation Equivalence:** For singular simple attributes (e.g., `userName`, `active`), Service Providers **MUST** treat `add` and `replace` as functionally equivalent.
2.  **Complex Sub-attribute Targeting:**  When only intending to modify the value of a specific sub-attribute of a complex attribute, Clients **SHOULD** target that sub-attribute using dot-notation (e.g., `path: "name.givenName"`).
3.  **Complex Attribute Operations:**
    *   **Replace:** If a Client targets a singular complex attribute in its entirety (e.g., `path: "name"`) using the `replace` operation, the `value` **MUST** be a JSON object containing all sub-attributes the Client intends to persist. The Service Provider **MUST** replace the entire complex attribute with the provided object.
    *   **Add:** If a Client targets a singular complex attribute in its entirety using the `add` operation, the `value` **MUST** be a JSON object. The Service Provider **MUST** perform a partial merge, updating only the provided sub-attributes and preserving existing values for omitted sub-attributes.

#### Multi-valued Attributes (Simple and Complex)
This section applies to both multi-valued simple attributes (e.g., the `schemas` attribute) and multi-valued complex attributes (e.g., `emails`, `addresses`).

1.  **Collection-Level Operations:**
    *   **Replace:** If a Client targets a multi-valued attribute path without a filter (e.g., `path: "emails"` or `path: "roles"`) using the `replace` operation, the `value` **MUST** be an array of objects/values. The Service Provider **MUST** replace the entire collection with the provided array.
    *   **Add:** If a Client targets a multi-valued attribute path without a filter using the `add` operation, the `value` **MUST** be an array of objects/values. The Service Provider **MUST** append the provided values to the existing collection.
2.  **Filtering Constraints:** When using a filter to target an element within a multi-valued complex attribute:
    * **Sub-attribute Requirement:** The `path` **MUST** target a specific sub-attribute of the matched element (e.g., `path: "emails[type eq \"work\"].value"`). Targeting the element object itself (e.g., `path: "emails[type eq \"work\"]"`) is **PROHIBITED**.

### Error Handling

Service Providers **MAY** return an HTTP `400 Bad Request` for any PATCH operation that violates these constraints. Specific `scimType` values should be used as follows:
* `invalidSyntax`: Missing `path` or use of prohibited operators.
* `invalidFilter`: Filter matches more than one element in a multi-valued attribute.
* `invalidPath`: Filter fails to target a specific sub-attribute.

## Resource Lifecycle

Service Providers **MUST NOT** treat a `PATCH` request that sets the `active` attribute to `false` as a deletion of the resource. A disabled user is considered inactive and should be excluded from authentication and normal access, but the user object itself **MUST** be retained by the Service Provider. Deletion of a resource can only be performed via an HTTP `DELETE` request.

## Concurrency and Versioning

SCIM clients **MUST NOT** include HTTP headers related to conditional requests or entity tags (ETags), such as `If-Match`, `If-None-Match`, `If-Modified-Since`, or `If-Unmodified-Since`. Service Providers are not expected to support these headers and **MAY** ignore them or reject the request. This profile relies on the principle of "last write wins" for simplicity.

## Bulk Operations

Support for the `/Bulk` endpoint ([RFC7644], Section 3.7) is RECOMMENDED butOPTIONAL.

## Error Handling

### Uniqueness Conflicts

When a `POST` or `PATCH` request attempts to create or modify a resource in a way that violates a uniqueness constraint (e.g., for attributes like `userName` or `externalId`), the Service Provider **MUST** return an HTTP `409 Conflict` response. The response body **MUST** also include a SCIM error detail with the `scimType` set to `uniqueness`, as defined in [RFC7644], Section 3.12.

# Security Considerations

## Transport Security

All communication between a SCIM client and service provider MUST be secured using Transport Layer Security (TLS) version 1.2 [RFC5246] or a later version.

## Authentication

The use of HTTP Basic Authentication over TLS is NOT RECOMMENDED.

# Deprecated Features

This section provides a summary of SCIM 2.0 features that are considered deprecated or out of scope for this profile to improve security and interoperability. Implementations conforming to this profile SHOULD NOT use the following features:

*  **HTTP Basic Authentication**: As stated in Section 6.2.
*  **Complex `PATCH` Syntax**: The following `PATCH` syntaxes are prohibited as defined in Section 5.4:
    * "Path-less" operations where the `path` attribute is omitted.
    * Path filters that do not resolve to a unique element.
    * Path filters using operators other than `eq`.
    * Path filters that target a whole element instead of a sub-attribute (e.g., `emails[type eq "work"]`).
*  **Certain Filter Operators**: Such as `pr` (presence).
*  **HTTP PUT**: The HTTP PUT method is deprecated in favor of HTTP PATCH to simplify implementation and reduce the risk of accidental data loss.
*  **User "password" attribute**: The "password" attribute is deprecated and MUST NOT be implemented. Service providers SHOULD NOT store user passwords and should rely on other authentication methods, such as federation via SAML or OpenID Connect, to authenticate users.
*  **Filters in URI for non-GET methods**: The use of filters in the URI for any HTTP method other than GET (e.g., PATCH /Users?filter=department eq 'sales') is deprecated and MUST NOT be used.
*  **Silent Ignoring of Unknown Attributes**: Service Providers MUST reject requests containing unknown attributes or schemas, as defined in Section 4.4.
*  **Client-side Versioning**: Clients MUST NOT send requests with HTTP versioning or ETag-related headers, as defined in Section 5.6.
*  **Disabling Users as Deletion**: Service Providers MUST NOT translate disabling a user (`active` = `false`) into a deletion, as defined in Section 5.5. 

# IANA Considerations

This document has no IANA actions.

# Acknowledgements

*(TODO: Add acknowledgements)*