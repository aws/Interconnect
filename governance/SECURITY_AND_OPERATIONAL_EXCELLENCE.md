# Security and Operational Excellence Policy

This policy establishes the security and operational excellence requirements for
the Connection Coordinator API specification, which enables cross-provider
direct connections between two network providers. As an openly licensed
specification under Apache 2.0, we encourage broad adoption and contribution
from the cloud provider community.

To maintain the integrity, security, and reliability of implementations, we have
established this policy to define minimum standards that all specification
changes must adhere to. This ensures that while the specification remains open
and adaptable, critical security and operational capabilities are preserved
across all implementations.

The primary goals of this policy are to:

1.  Protect customer data traversing cross-provider connections
1.  Ensure consistent security practices across all implementing cloud providers
1.  Maintain operational reliability for mission-critical workloads
1.  Provide clear guidance for contributors proposing specification changes

All proposed changes to the specification must comply with these requirements.
The project Maintaining Organizations will evaluate contributions against these
standards to preserve the security-first design and operational excellence
principles that are foundational to this initiative. Per the Governance
Framework, this policy will be reviewed annually to ensure that these tenets are
still necessary to achieve the project’s performance and security objectives.

## Core Security Tenets

### Mandatory Encryption

-   No changes that make encryption optional or weaken encryption standards will
    be accepted.
-   TLS 1.3+ must be used for all API communications between providers.
-   Connection features that support encryption (like MACsec) must remain
    mandatory where specified.

### Authentication and Authorization

-   All API endpoints must implement authentication mechanisms that follow
    encryption and security best practices, such as AWS’ Well-architected
    recommendations and in particular the [Security and Operational Excellence
    Pillars](https://aws.amazon.com/architecture/well-architected/) and [Google’s
    Well-Architected Framework: Security, privacy, and compliance pillar](
    https://docs.cloud.google.com/architecture/framework/security).
-   This specification does not outline a specific authentication and
    authorization API. Instead we expect providers to agree on how the best
    practices above will be met with their existing platforms.
-   The specification must maintain a clear mapping from the chosen
    authorization framework to individual providers defined within the
    specification. Implementations must prevent unauthorized access to a
    provider's resources.
-   The security sections in API paths must not be emptied or weakened.
-   Activation key validation must remain a required step in connection
    establishment.
-   Providers are responsible for ensuring API requests (Create, Delete, etc)
    affecting customer owned resources (Connections, Features, etc) reflect the
    intent of the customer. The specification includes specific mechanisms to
    help validate and avoid confused deputy vulnerabilities across providers.

### Connection Integrity

-   Activation keys are one time use by the customer. After the key is used
    successfully to activate capacity, this key is no longer valid for creating
    any future connections. Note, the activation keys are preserved with the
    connection resource for informational purposes only.
-   Activation keys must be generated using UUIDv4 to prevent key collisions
    across customers.
-   Activation key encoding must use BASE64 encoding to ensure deterministic
    decoding.
-   No changes that bypass activation key validation will be accepted, unless
    replaced with an objectively stronger mechanism.

### Secure Channel Management

-   The MACsec manager provider role must be maintained to ensure proper
    encryption key management.
-   MACsec keys must be rotated once every 30 days.
-   Only one channel within an interconnect can have it's MACsec key's rotated
    at any given time.
-   Channel security features must not be removed.

## Operational Excellence Tenets

### Service Reliability

-   Maintenance event notifications must remain part of the specification.
-   When performing elective maintenance, a provider is responsible for ensuring
    their maintenance does not overlap with an existing maintenance on any other
    channels within an interconnect. If through a race condition, two providers
    schedule concurrent maintenance, both providers are expected to reschedule.
-   Providers may perform maintenance on the same channel simultaneously.  This
    is important to account for facility downtime.
-   Connection status monitoring capabilities must be preserved.

### Resource Management

-   Available capacity must be accurately reported on the interconnect as the
    amount of connection capacity that provider can still support provisioning
    in that moment.
-   Connection resource cleanup requirements (e.g., 7-day deletion policy) must
    be maintained.

### Fault Isolation

-   Channels must be deployed across diverse routers when contained within a
    single interconnect. This diversity must be maintained after provisioning.
-   The obfuscated device ID must reflect a unique ID for the routers hosting
    the channel.
-   No changes that would compromise redundancy capabilities will be accepted.

## Implementation Requirements

### Specification Compliance

-   Any additional API features are considered optional, but if implemented,
    must be implemented in full.
-   Implementations must maintain all required fields marked as such in the
    specification.

### Feature Compatibility

-   All implementations must support the FEATURE_TYPE_L3_BASE feature type. All
    other feature types can be considered optional.
-   No changes that would break interoperability between providers will be
    accepted.
-   All features must remain compliant with their relevant RFCs (Example: BGP:
    RFC4271). We will not accept configuration constructs that do not adhere to
    these specifications.

## Change Management Policy

### Security Impact Assessment

-   Any steering committee member can request a security impact assessment for a
    specification change.
-   Changes that reduce security capabilities will be rejected.
-   Security considerations must take precedence over implementation simplicity.

### Backward Compatibility

-   Changes must maintain backward compatibility where possible.
-   Breaking changes must provide clear migration paths.
-   Version management must follow semantic versioning principles.

### Documentation Requirements

-   All changes must include appropriate documentation updates.
-   Security implications must be clearly documented.
-   Implementation guidance must be provided for security-critical features.
