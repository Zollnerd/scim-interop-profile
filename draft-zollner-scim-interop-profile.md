---
title: SCIM 2.0 Interoperability Profile
abbrev: SCIM Interop Profile
ipr: trust200902
category: std
area: Applications and Real-Time
workgroup: SCIM
keyword:
  - scim
  - provisioning
  - identity

pi: [toc, sortrefs, symrefs]

docname: draft-zollner-scim-interop-profile-latest

author:
  -
    name: Danny Zollner
    organization: Okta
    email: danny.zollner@okta.com

normative:
  RFC7644:
  RFC7643:
  RFC9865:
  RFC9325:
  RFC6585:
  RFC9110:

---


--- abstract

This document defines an implementation profile for the System for Cross-domain Identity Management (SCIM) 2.0. In the typical deployment model, an identity provider acting as a SCIM Client provisions and manages identities at multiple downstream service providers, while each service provider (commonly a multi-tenant application serving many customers) accepts provisioning connections from multiple different identity providers. This many-to-many integration model compounds the interoperability challenges arising from the wide range of optional features and implementation variations permitted by the base specification. This profile addresses those challenges by restricting the optional features and protocol variations permitted under the base specification and by establishing normative requirements for a common implementation baseline.

--- middle

# Discussion Venues

This note is to be removed before publishing as an RFC.

Source for this draft and an issue tracker can be found at https://github.com/Zollnerd/scim-interop-profile.

# Introduction

The SCIM 2.0 standard [RFC7643] [RFC7644] provides a framework for automating identity provisioning across organizational boundaries. While the base specification's flexibility enables broad applicability, it also permits a wide range of implementation choices — in areas such as `PATCH` syntax, pagination, filtering, case sensitivity, and error handling — that have proven to be common sources of interoperability failure in practice. Where implementations diverge on these points, the result is integrations that require bespoke configuration and maintenance for each pairing.

This document specifies a profile for SCIM 2.0 that addresses these challenges by establishing required capabilities and restricting protocol variations that have proven problematic. Implementations conforming to this profile can be expected to interoperate without integration-specific customization.

# Notational Conventions

{::boilerplate bcp14-tagged}

# Scope and Conformance

This profile applies to SCIM 2.0 Service Providers and Clients.

Absolute requirements and prohibitions in this document, expressed using the BCP 14 keywords "MUST", "MUST NOT", "SHALL", "SHALL NOT", and "REQUIRED" as defined in the Notational Conventions section above, are addressed to Service Providers, Clients, or both. A Service Provider is conformant with this profile if it satisfies all such requirements addressed to Service Providers and all such requirements addressed to both roles. A Client is conformant with this profile if it satisfies all such requirements addressed to Clients and all such requirements addressed to both roles.

Service Providers claiming conformance with this profile MUST include an `interopProfileConformant` attribute with a value of `true` in their `ServiceProviderConfig` response. This attribute is defined as follows:

* Name: `interopProfileConformant`
* Type: Boolean
* Multi-Valued: false
* Required: false
* Mutability: readOnly
* Returned: default

A Service Provider that does not conform to this profile MUST either omit this attribute or set its value to `false`.

# Data Model Requirements

## Discovery Endpoints

Service Providers MUST implement the following configuration discovery endpoints:

*  `ServiceProviderConfig` ([RFC7643], Section 4)
*  `Schema` ([RFC7643], Section 7)
*  `ResourceType` ([RFC7643], Section 6)

Service Providers MUST publish an accurate list of schemas and attributes via the `/Schemas` endpoint, matching exactly what is implemented and supported by the service. All schemas referenced in resource type definitions returned by the `/ResourceTypes` endpoint MUST be available at the `/Schemas` endpoint.

The schema and resource type definitions for `ServiceProviderConfig`, `Schema`, and `ResourceType` MAY be omitted from the `/Schemas` and `/ResourceTypes` endpoints respectively, as these are configuration resources rather than provisioning targets.

For every `ResourceType` resource returned by the `/ResourceTypes` endpoint, Service Providers MUST populate the `id` attribute and its value MUST equal the value of the `name` attribute.

## Case Sensitivity

To ensure predictable and interoperable behavior, Service Providers **MUST** implement case sensitivity consistently across both filtering operations and uniqueness constraint enforcement. For any given attribute, the case sensitivity rules applied during a filter query (e.g., `userName eq "User.A"`) **MUST** be identical to the rules used to detect a uniqueness conflict during a `POST` or `PATCH` operation.

This profile requires adherence to the case sensitivity definitions specified in [RFC7643] for the following attributes:

Common Attributes ([RFC7643], Section 3.1):
*   `id`: `caseExact` is `true`. Service Providers **MUST** treat `abc-123` and `ABC-123` as distinct values.
*   `externalId` (optional): `caseExact` is `true`. Service Providers **MUST** treat `ABC-123` and `abc-123` as distinct values, if `externalId` is supported.

User Schema Attributes ([RFC7643], Section 4.1.1) -- applicable only if the User resource type is implemented:
*   `userName`: `caseExact` is `false`. Service Providers **MUST** treat `JSmith` and `jsmith` as equivalent.
*   `emails.value`: `caseExact` is `false` ([RFC7643], Sections 4.1.2 and 8.7.1). Service Providers **MUST** treat `user@example.com` and `USER@EXAMPLE.COM` as equivalent.

## Attribute and Schema Handling

To ensure strict conformance and prevent unintended data loss or corruption, Service Providers **MUST** adhere to a strict handling model for attributes and schema extensions. When a SCIM request (e.g., `POST`, `PATCH`) contains attributes or schema URIs that are not defined in the Service Provider's `ResourceType` or `Schema` definitions, the request **MUST** be rejected.

The Service Provider **MUST** return an HTTP `400 Bad Request` with a `scimType` error of `invalidValue` for such requests. Clients **MUST** consult the `/Schemas` endpoint and **MUST NOT** send attributes or schema URIs that the Service Provider does not advertise there.

## Canonical Values for Typed Attributes

For multi-valued complex attributes that include a `type` sub-attribute (e.g., `emails`, `phoneNumbers`, `ims`, `addresses`), Service Providers MUST declare the acceptable `type` values for each such attribute in that attribute's `canonicalValues` property, as returned by `/Schemas`. Clients MUST only use `type` values listed in the `canonicalValues` property for that attribute.

# Protocol and Endpoint Requirements

## Endpoint Structure

Service Providers MUST offer a unique endpoint for each implemented resource type (e.g., `/Users`, `/Groups`). Resources of different types MUST NOT share an endpoint. This requirement applies only to resource-type endpoints; the root endpoint (e.g., `https://example.com/scim/v2/`) is not subject to this requirement and MAY return resources of multiple types if querying at the root level is supported.

Service Providers MUST use `/{endpoint}/{id}` as the canonical URI for addressing any resource and MUST NOT address resources via other attribute values in the URI path (e.g., `/{userName}`). Clients MUST use `/{id}` when retrieving a known resource, and MUST use the `filter` query parameter to locate a resource by any other attribute value.

## Data Format and HTTP Headers

All data exchange MUST use the JSON format as defined in [RFC7643]. All requests and responses containing SCIM data MUST include a `Content-Type` header with the value `application/scim+json`, as defined in [RFC7644], Section 8.1.

To aid in troubleshooting and client identification, Clients MUST include a `User-Agent` header in all HTTP requests. The header's value should be meaningful, for example, identifying the name of the client software.

## Filtering

The `filter` query parameter MUST be supported. Service Providers MUST support the `eq` and `and` operators. To promote interoperability, Clients MUST NOT use operators other than `eq` and `and`, whether the filter is sent in the `filter` query parameter of a `GET` request or in the body of a `POST` query to `/.search` ([RFC7644], Section 3.4.3). Service Providers MUST support filtering with the `eq` operator on every attribute for which they enforce a uniqueness constraint (e.g., `userName`), including when that attribute is a sub-attribute of a multi-valued attribute (e.g., `emails.value`), so that Clients can identify the resources involved in a uniqueness conflict.

Clients **MAY** use filters in read operations: `GET` requests using the `filter` query parameter, and `POST` queries to the `/.search` endpoint with the filter in the request body. Clients **MUST NOT** include a filter in the HTTP request URI of a request that uses any HTTP method other than `GET` (e.g., `PATCH /Users?filter=...`). This restriction applies only to the HTTP request URI. It does not apply to a filter in the `path` attribute of a `PATCH` operation ([RFC7644], Section 3.5.2) or to a filter in the body of a `POST` query to `/.search`.

When a filter expression is applied to a resource-type endpoint (e.g., `/Users`, `/Groups`) and references an attribute not defined in any of that resource type's schemas, Service Providers MUST return an HTTP 400 error with a `scimType` of `invalidFilter`, as defined in [RFC7644], Section 3.12.

## Pagination

To ensure the reliable handling of large data sets, Service Providers MUST implement at least one of the following pagination methods for all list operations:

* Index-based pagination as defined in [RFC7644], Section 3.4.2.4.
* Cursor-based pagination, as defined in [RFC9865].

Clients MUST support both index-based and cursor-based pagination. When a response contains `nextCursor`, Clients MUST continue retrieving results using it, including when the Client did not send a `cursor` parameter ([RFC9865], Section 2.3). Clients MUST NOT treat a response as the final page unless it indicates so under the pagination method in use.

Service Providers MUST include the `pagination` attribute defined in [RFC9865], Section 4 in their `ServiceProviderConfig` response, including the `cursor` and `index` sub-attributes. A Service Provider that receives a pagination query parameter (`startIndex` or `cursor`) for a method it does not support MUST return an HTTP `400 Bad Request` error and MUST NOT ignore the parameter.

If the `count` parameter is omitted from a request, Service Providers SHOULD return at least 100 results by default. Service Providers MUST support a client-requested `count` value of at least 100. Service Providers SHOULD enforce a server-specified maximum number of results per page and MUST truncate to that maximum, rather than returning an error, when the client specifies a `count` value that exceeds it.

## Updating Resources

Service Providers MUST support the `PATCH` operation ([RFC7644], Section 3.5.2) for resource updates. Clients MUST use the `PATCH` operation for updates and SHALL NOT use the `PUT` operation.

This profile defines a restricted subset of the SCIM 2.0 `PATCH` method to ensure predictable behavior and high interoperability.

### General PATCH Constraints

#### Mandatory Use of the 'path' Attribute
Every operation object within the `Operations` array **MUST** contain a `path` attribute. Clients **MUST NOT** issue "path-less" PATCH operations where the target attribute is implied by the keys within the `value` object.

Service Providers **MAY** reject PATCH requests containing operations that lack a `path` attribute. A Service Provider that rejects such a request **MUST** return an HTTP `400 Bad Request` and a `scimType` error of `invalidSyntax`.

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
    * **Sub-attribute Requirement:** The `path` **MUST** target a specific sub-attribute of the matched element (e.g., `path: "emails[type eq \"work\"].value"`). Clients **MUST NOT** target the element object itself (e.g., `path: "emails[type eq \"work\"]"`).
    * **Operator Restriction:** Clients **MUST NOT** use filter operators other than `eq` and `and` in a `path` value (e.g., `emails[type co \"work\"].value`).
    * **Single-Match Requirement:** Clients **MUST NOT** use a filter in a `path` value that matches more than one element of the multi-valued attribute.

### PATCH Error Handling

A Service Provider that rejects a PATCH operation violating the constraints defined in this section **MUST** return an HTTP `400 Bad Request` and **MUST** use the `scimType` values as follows:

* `invalidSyntax`: Missing `path`.
* `invalidFilter`: Filter uses an operator other than `eq` and `and`.
* `invalidPath`: Filter fails to target a specific sub-attribute, or filter matches more than one element in a multi-valued attribute.

## Resource Lifecycle

Service Providers **MUST NOT** treat a `PATCH` request as a deletion of the resource. A resource modified via `PATCH` **MUST** be retained by the Service Provider. Deletion of a resource can only be performed via an HTTP `DELETE` request.

Service Providers MUST return `404 Not Found` in response to any operation targeting a deleted resource and MUST omit deleted resources from all query results.

After a resource is successfully deleted, its unique identifiers (such as `userName`) MUST NOT be considered in uniqueness conflict calculations and MUST be available for reassignment to a new resource.

Service Providers MUST NOT respond to an HTTP `DELETE` request by modifying resource attributes rather than deleting the resource (e.g., setting a User's `active` attribute to `false`).

## Concurrency and Versioning

Clients **MUST NOT** include HTTP headers related to conditional requests or entity tags (ETags), such as `If-Match`, `If-None-Match`, `If-Modified-Since`, or `If-Unmodified-Since`. Service Providers are not expected to support these headers. A Service Provider that does not evaluate a conditional request header **SHOULD** reject the request with HTTP `412 Precondition Failed` rather than ignoring the header, since a Client that sends such a header expects it to be enforced.

This requirement means the profile provides no protection against lost updates when more than one Client modifies the same resource, including multiple identity providers provisioning to the same Service Provider. This is a deliberate trade-off in favor of keeping both Clients and Service Providers simple. Because every `PATCH` operation must include a `path`, a Client's update affects only the attributes it targets.

## Bulk Operations

Support for the `/Bulk` endpoint ([RFC7644], Section 3.7) is OPTIONAL.

## Error Handling

### Uniqueness Conflicts

When a `POST` or `PATCH` request attempts to create or modify a resource in a way that violates a uniqueness constraint (e.g., for attributes like `userName` or `emails`), the Service Provider **MUST** return an HTTP `409 Conflict` response. The response body **MUST** also include a SCIM error detail with the `scimType` set to `uniqueness`, as defined in [RFC7644], Section 3.12.

### Rate Limiting

Service Providers that limit the rate of requests SHOULD respond with HTTP `429 Too Many Requests` ([RFC6585], Section 4) and SHOULD include a `Retry-After` header ([RFC9110], Section 10.2.3) indicating how long the Client should wait before retrying. A Service Provider MAY instead respond with HTTP `503 Service Unavailable` ([RFC9110], Section 15.6.4), which MAY also include a `Retry-After` header.

When a Client receives a `429` or `503` response, it MUST NOT retry the request before the time indicated by the `Retry-After` header has elapsed. If the response does not include a `Retry-After` header, the Client MUST apply a backoff before retrying.

# Security Considerations

## Transport Security

All communication between a Client and Service Provider MUST be secured using Transport Layer Security (TLS). Implementations MUST follow the recommendations in BCP 195 [RFC9325].

## Authentication

The use of HTTP Basic Authentication over TLS is NOT RECOMMENDED.

## Resource Addressing

This profile requires that resources be addressed by `id` rather than by other attribute values in the URI path. If a resource could also be addressed by a mutable attribute such as `userName`, the value of that attribute could be transferred between resources, and a request addressed to the old value would modify a different resource than the Client intended. For example, User A holds the `userName` "alice" and User B holds the `userName` "bob". If those values are swapped, so that A now holds "bob" and B now holds "alice", a Client issuing `PATCH /Users/alice` with entitlements or role assignments intended for the original User A would instead modify User B. The `id` attribute is assigned by the Service Provider and is stable and non-reassignable ([RFC7643], Section 3.1), which removes this risk.

## Identifier Reuse

Resources have two kinds of identifiers. The `id` attribute is assigned by the Service Provider, is immutable, and is non-reassignable ([RFC7643], Section 3.1). Attributes such as `userName` and `emails.value` are subject to uniqueness constraints but are set by Clients and can usually be changed. Their values can also be reused over time. For example, the `userName` "jsmith" can belong to one person and later, after that person's resource is deleted, to a different person.

A Service Provider that supports the HTTP `DELETE` method makes the unique identifiers of a deleted resource, such as `userName`, available for reassignment to a new resource, as described in the Resource Lifecycle section. Because the values of these attributes can change and be reused, Clients, Service Providers, and other systems SHOULD NOT use them as stable references to a resource, for example as the key for access grants or role assignments. They SHOULD use `id` for that purpose. A record keyed on a reusable value can end up applying to a different person than the one it was created for.

A Service Provider that does not want these values to be reused SHOULD NOT permit the resource to be deleted, and SHOULD instead retain the resource with the `active` attribute set to `false`. The `active` attribute is a Boolean and cannot distinguish a temporarily disabled resource from one that is permanently disabled and retained to prevent reuse of its identifiers.

# IANA Considerations

Prior to being published as an RFC, this document requests that the IANA SCIM Server-Related Schema URIs registry entry for `urn:ietf:params:scim:schemas:core:2.0:ServiceProviderConfig` be updated to include or reference the `interopProfileConformant` attribute defined in Section 4 of this document.

# Acknowledgements

*(TODO: Add acknowledgements)*
