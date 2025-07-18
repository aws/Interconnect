# MultiCloud Management API: Object Types

This document is meant to be a supplement to the OpenAPI specification.  It's objective is to provide some clarity around the overall object hierarchy and what each item actually represents.

## Provider
The base type of `provider` is used to delineate the respective CSP from which requests are being made.  For example, as Amazon, we would make requests toward partners using `/v1/providers/aws` as our prefix.  This is used as a way of ensuring resource level permissions and isolation server side.

## Environment
The `environment` object is used to uniquely describe connectivity between two partner CSPs into specific regions.  This is something that the customer would select when creating their connection and would distinctly identifiy the two regions that are being connected.

For example, a single `environment` could describe the generic path between (AWS us-east-1) and (GCP us-east-4a) and another could describe a path (AWS us-east-1) and (AZURE East US).

## Interconnect
The `interconnect` object describes a bundle of `channels` upon which the CSP would place customer `connections`.  Each `interconnect` exists as part of an `environment` and each `environment` may have many interconnects.  Customers never see the `interconnect` objects, they are solely used by the CSPs.

The `interconnect` objects reflect the pre-cabled capacity that will be sold to customers and reserved for their use.

## Channel
The `channel` object is a single LAG that resides as part of an `interconnect`.  There are 4 `channels` per `interconnect`, and each one is a LAG comprising the same number of 100G (or 400G, but must be uniform) ports, resulting in identical capacity on all `channels`.

## Connection
The `connection` object represents an explicit customer request for a specific bandwidth across a specific `environment`.  After a customer requests a `connection`, our APIs will be used to negotiate the specific `interconnect` to place this `connection` on.  The customer has no influence over the `interconnect` choice.

## Feature
The `feature` object is an extensible object to describe anything that we may want to support on a given connection.  At this time, there is only one feature type, which is Channel Config.

The Channel Config `feature` is used to convey a negotiated set of L3 parameters including VLAN, IPv4 Subnet, IPv6 Subnet, and MTU.  Each of these parameters is negotiated as part of the customer's creation.  Once again, the customer has no real influence over the chosen parameters as they are distinctly negotiated between the two CSPs.

When this Channel config is agreed upon, a hosted connection of the appropriate bandwidth is created on the LAG represented by the `channel`.


