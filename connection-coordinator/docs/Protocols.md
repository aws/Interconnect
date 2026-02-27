# **Connection Coordinator API: General Protocols**

## **API Summary**

The Connection Coordinator API is designed to allow for two partner providers to
coordinate peered connections into their respective networks via a managed
service. The APIs have been designed in a way to allow for maximum flexibility
in terms of how the specific details should be organized between the two
partners. Despite this flexibility, there are still certain expectations that
need to be followed in order to be truly compliant with the API specification.
Generally, we will refer to these expectations as the protocols that should be
followed when leveraging the APIs in a proper manner.

The API is implemented as a fully REST based implementation governing the
lifecycles of a few discrete objects, which are described below. These
lifecycles also dictate elements of the protocols which will govern how the
objects should be treated when reflecting their configurations against the
backing network hardware.

## **Authentication**

API calls made into a partner are expected to leverage industry standard
encryption schemes as well as adhere to the authentication and authorization
expectations of that partner. For example, to make calls into AWS, partners
will be expected to:

*   Procure AWS accounts in good standing.
*   Inform AWS of the account numbers that should be marked as an AWS
    Interconnect partner
*   Use AWS SigV4 authentication on all inbound requests.

API calls are also marked with additional information governing the specifics
of their use. One such example is requests that are marked as requiring
[Confused Deputy Resolution](https://quip-amazon.com/U4jIAxNo1B1m/Connection-Coordinator-API-General-Protocols#temp:C:deH3724743bac954f07b8c8ecefb)
— callouts such as this one are not mere suggestions and are considered a
minimum security requirement for proper API utilization.

## **API Object and Relationships**

### **Provider**

The **Provider** object is at the root of the API hierarchy. It is used to
indicate the respective Partner from which requests are being made. This is
used as a way of ensuring resource level permissions and isolation server side.
Each participating **Provider** is assigned a generally accepted identifier of
their choosing (within reasonable naming schemes). For example, as AWS, we
would make requests toward partners using `/providers/aws` as the URI prefix,
and GCP would instead be making the requests using `/providers/gcp` .

This object’s primary focus in the API is to serve as a resource delineation
point. If the service you implement is going to be interacting with multiple
partners, then having the `/providers` distinction upfront will make resource
separation a much easier task. Additionally, the **Provider** object also
offers a way for each partner to provide information that will be helpful in
the user experience of the solution, such as getting the desired display name
for a partner. This would also be the node at which additional context is
provided around
[User Cross-Cloud Authentication](https://quip-amazon.com/U4jIAxNo1B1m/Connection-Coordinator-API-General-Protocols#temp:C:deH395aad7d2e82428191348bb0c).

### **Environment**

The **Environment** object is best thought about as a way to describe the
physical locations that map to the connectivity being provided. Generally
speaking, the expectation is that most partners will want to create peered
connections by leveraging buildings and networking infrastructure which is
already in close proximity. That said, there is no real restriction placed on
how two partners decide to leverage the **Environment**.

In its simplest form, an **Environment** can be thought of as a pair of Partner
Locations, one for each **Provider**. For example, AWS and GCP have created an
environment that connects **AWS us-east-1** to **GCP us-east4**. When creating
these environments, both partners need to agree upon the shared Environment
ID. Continuing on the previous example, let’s say AWS and GCP agreed to call
the environment `aws-gcp-us-east` . In that case, the respective URI paths
would take the respective forms:
AWS: /providers/aws/environments/aws-gcp-us-east
GCP: /providers/gcp/environments/aws-gcp-us-east
It is imperative that both providers agree upon and leverage the same
Environment ID to allow for coordination of the connections that customers
will be looking to create within this environment. It is also worthwhile to
note that this value should be generally unique across all participating
partners environments — while this is not a requirement, it does make customer
selections easier to manage.

The major takeaway of an **Environment** object is that this is effectively
the major choice the customer would be selecting when they are deciding that
they want to create a connection—the general locations that are being
connected. While the **Environment** object is laying a logical construct that
gives the customer visibility into which partners and which locations will be
connected for maximum efficiency, the **Environment** merely summarizes the
underlying available bandwidth present on the **Interconnect** child objects..

### **Interconnect**

The **Interconnect** object stands as a logical object that represents an
effective grouping of physical connectivity. Essentially, the **Interconnect**
exists as a redundancy group of **Channel** objects across which the
customer’s **Connection** will be built. The primary function of the
**Interconnect** is to serve as a horizontal scaling mechanism underneath an
**Environment**. If we run out of capacity within an **Environment**, we would
need to either expand an existing **Interconnect** or create a new one.

The general idea behind using the **Interconnect** object rather than going
directly to the **Channel**, is that the **Interconnect** provides the
guaranteed redundancy that is required for proper operation of a production
grade implementation.

Similar to the **Environment** object, the **Interconnect** must be assigned
an agreed upon name by both partners to ensure that API calls can refer to the
same objects. Unlike the **Environment**, the **Interconnect** name only need
uniqueness within the **Environment** scope. For example, calling an
**Interconnect** `interconnect1` or even just `ix1` is completely reasonable
— in fact, a simple naming scheme here is preferred as most resources already
have an entire URI path that will describe them, so any additional information
is likely to be redundant.

### **Channel**

The **Channel** object is merely a generic term to describe a network path
between two network nodes owned by the respective partners. In practice, the
general expectation is that this object is effectively a stand-in for a
Physical LAG of similar posture on each side of the network.

Naming wise, **channel** objects also need an agreed upon name — the
recommendation is to keep it simple with `channel1`or `ch1` . Production grade
solutions should have no fewer than four **channel** objects per
**interconnnect**. Take note, the API provides ways to perform migrations which
may lead your runtime solutions to adding new channels in the long run and
deprecating some of the older ones — it is for that reason that even though
the initial configuration of the channel objects will lend itself nicely to
iterating over the channels from 1 to 4, such iterations should be avoided to
be able to handle all required API functions and protocols.

### **Connection**

The **connection** object represents an explicit customer request for a
specific bandwidth across a specific **environment**. After a customer requests
a **connection**, the connection coordination APIs will be used to negotiate
the specific **interconnect** on which to place this connection. The customer
has absolutely no influence over the **interconnect** choice as this is part of
the managed service that we are providing to the customer.

Our promise to the customer is that they will get a **connection** of specified
bandwidth that spans the chosen **environment**.

### **Feature**

The **feature** is an extensible object to describe anything that we may want
to support on a given **connection**. At this time, there is only one feature
type, which is a “channel config”.

The channel config feature is used to convey a negotiated set of L3 parameters
including ASN, VLAN, IPv4 Subnet, IPv6 Subnet, and MTU. Each of these
parameters are negotiated as part of the customer's creation using the
[Negotiation Protocols](https://quip-amazon.com/U4jIAxNo1B1m/Connection-Coordinator-API-General-Protocols#temp:C:deH811e42b072714852b322f414b).
Once again, the customer has no influence over the chosen parameters as they
are distinctly negotiated between the two partners—the only exception being
the ASN, which may be derived from the customer's existing network details.

## **Static Resources**

At the time of writing, static resources (provider, environment, interconnect,
and channel) are managed outside of the API specification. Both providers must
agree on the attributes and resource names for each during the physical
provisioning process. These resources are then available to be queried by
either provider to verify correctness and used to create dynamic resources such
as connections and features.

Note: In the future we may add support for mutating these resources to the API
spec.

## **Customer Connection Workflow**

When a customer is looking to make a connection, there are 3 milestones that
must be reached:

1.  Establish Customer Intent
2.  Negotiate Connection/Feature Parameters
3.  Provision Appropriate Features

### **Establish Customer Intent**

The first and most important milestone is to establish the customer’s intent
to create a cross cloud connection, especially since both partners need to
ensure that the intent exists to avoid security concerns. This intent is
established using a Create operation on one provider’s portal followed by an
Accept operation on the other partner’s portal.

The customer goes to the first partner using an authenticated account to invoke
a Create operation. This Create operation captures the customer’s intent by
selecting the specific **Environment** and a desired bandwidth. The customer
must also supply the Account Identifier that they intend to connect into on the
other provider’s portal. The local provider MUST perform some perfunctory
checks, such as ensuring that the selected environment has the requested
bandwidth available, and that the customer’s account is in good standing and
fulfilling this request won’t exceed any quotas, etc.

These details are captured into an **ActivationKey**, along with a **Connection
UUID** to identify this proposed connection, which is to then be delivered by
the customer out of band to the partner provider. The remote provider
(receiving the activation key) needs to confirm the following:

*   The **ActivationKey** is syntactically valid.
*   The **Environment** specified is a valid selection
*   The desired bandwidth is in fact still available
*   That the listed account identifier in the **ActivationKey** is the same as
    the authenticated entity (or that the authenticated entity has the
    necessary permissions to act on behalf of that account identfiier).

After those checks are done on the remote provider, they call the
**ConfirmActivationKey API** to ask the original provider if this activation
key is valid for the specific **Connection UUID**. The remote provider must
verify:

*   The connection UUID references a resource that still exists,
*   The key has not already been used,
*   The requested bandwidth and environment match the customer’s intent.

If the key does not match the expectations, then the remote provider MUST NOT
proceed with making the connection (and SHOULD delete any temporary resource
around it). If the key is valid (meaning that it is an exact match for the key
given out by the first provider), then the Customer’s Intent is considered
established, and we can move on to the next stage.

### **Negotiate Feature Parameters**

With the intent established, we next need to align on a few connection
specific details. We make these assessments by using a negotiation protocol. In
general, the protocol follows a negotiator/responder pattern—the negotiator
makes a proposal, and the responder either accepts that proposal in full, or
rejects it in full. By default, the provider that received an **ActivationKey**
from the customer (and invoked the **ConfirmActivationKey API**) is assigned
the negotiator role.

The first parameter that needs to be negotiated is the selection of
**Interconnect**. At this point, while we have established the customer’s intent
to create a **Connection** within a specific **Environment** we have not
selected which to **Interconnect** on which the **Connection** should be placed.
The mechanism by which the **Interconnect** is proposed is simply by calling
the **CreateConnection API**, which effectively places the **Connection** as a
child of a specific **Interconnect**. This operation is a bit strange, because
we have already agreed upon the connection UUID (shared as part of the
activation key), but this Create operation allows us to place the object
appropriately underneath the correct **interconnect**. The selection logic here
is not complicated — the **Interconnect** that has the most available bandwidth
should be selected. The responder in this case should avoid rejecting the
proposal unless there simply isn’t space. Since these are shared resources,
the only way the connection wouldn’t fit is if there was a race between two
separate connections.

Once the **Interconnect** has been selected, that will enable us to move on to
the next Negotiation step, which is to align on the **feature** parameters.
The negotiator starts the process off by requesting **FeatureGuidance** from
the responder. This gives the responder the opportunity to have some control of
the inputs to the proposal process. The guidance is an operation scoped to the
**Connection** object, but discrete guidance should be given for each implicit
feature that will be made. Typically, the implicit feature set will be a
single L3 Base Config feature placed upon each of the **Channel** objects on
the selected **Interconnect**. Using this guidance, the negotiator then starts
making proposals for the **Feature** objects, by making **CreateFeature**
**API** calls. Once again, the responder either accepts the Feature as proposed
(200 OK) or rejects it with some HTTP 400 error. There are two primary cases
for rejection. One case is that the request is simply wrong, either the
guidance wasn’t followed, or a parameter is simply unacceptable. The other
plausible case would be from a race in which another **Feature** managed to
get created in that space, leading to a conflict (HTTP 409).

Once all of the necessary **Feature** objects have been accepted with a 200 OK,
we are able to move onto the provisioning phase. A connection is considered
ready for provisioning when at least 3 out of 4 features have been
successfully negotiated.

### **Provision Appropriate Features**

The provisioning phase allows each partner to finish any and all steps
necessary to bring up the agreed upon **Features**, in particular the BGP
sessions. Since each partner has different networking requirements and
workflows, this is strictly a process handled by each side individually.
Additionally, knowing that each network has additional complexities that need
to be managed, each partner MUST rely on a local check to indicate availability
**and** on the notifications received on the **Connection** state before
verifying it for customer use.

Once all of the **Features** on a **Connection** become available, each partner
should send a **NotifyConnectionStatus** message—once a partner has sent this
notification (with acknowledgment) and received one from the other side, then
we can consider the **Connection** as Verified.

## **Update Connection Workflow**

The ability to update an existing **connection** is an important aspect of the
solution. This allows customers to seamlessly increase (or decrease) their
bandwidth as the business needs change. At present time, connection updates
are restricted to bandwidth only but similar changes would be done in a similar
manner.

When a customer requests a bandwidth change, the local provider has to make the
initial assessment to determine if the requested bandwidth is available on the
**Environment** at large. Even if the customer’s **Connection** is on an
**Interconnect** which itself doesn’t have space, we can still provide the
update via a **Migration**. If the bandwidth is available on the same
**Interconnect** that the **Connection** is already placed on, then we can
proceed with an in-place update. Prior to any network side changes, the update
must be confirmed with the peer provider using the **UpdateConnection**
operation. Changes are only made if the partner responds with a 200 OK.

In general, a partner may reject an update for any particular reason including
quota or limit related reasons. Additionally, if the partner is already
processing an update request from the customer, it should reject the new
request from the provider with a 409 Conflict (to avoid race conditions).

Upon agreeing to grant the update at both sides, we can immediately move to
re-provision the network components. Similar to when we were creating a
**connection**, a **NotifyConnectionStatus** message should be sent when all
changes are complete to indicate completion of the update to the other side.

## **Delete Connection Workflow**

The delete workflow is relatively straightforward. A customer’s intent to
delete should be enacted almost immediately, but similarly that intent should
be shared with the partner. The **DeleteConnection** operation should be sent
to indicate the customer’s desire to the other provider. This isn’t really an
operation that can be declined—there are possible exceptions such as ambiguous
customer intent (see Confused Deputy Resolution) or expectations around route
preferences.

It is important to recognize that a **DeleteConnection** operation comes with
an implied deletion of all underlying **Features** that are considered implicit
to the **Connection**, thus preventing the need for fully sequenced delete
coordination when the customer’s intent was made clear from the outset.

## **Delete Feature**

As this is a managed service, there will be times when a given **Feature**
simply needs to be deleted. This could be for a number of reasons such as:

*   The customer has decided to remove a non-implied **Feature**
*   We need to move to new hardware in a new physical building.

Note, that as previously mentioned, a **DeleteConnection** operation is not
required to be preceded by 4 **FeatureDelete** calls. The deletion of those
**Features** would be implied as part of the **DeleteConnection.**

In the case where a **FeatureDelete** is considered appropriate, it is
recommended that the initiating provider precede such a request by first
depreferencing the routes to/from the respective Channel. (More to come on
this later).

Once a **Feature** has been deleted, the feature will be eventually recreated
to fulfill the connection. Each **Channel** is assigned an owner that is
responsible for ensuring that any missing features are added to eventually
converge on a full set of features.

## **Migration Workflow \[In Progress\]**

We have the ability to Migrate a Connection from one of the Interconnects
within a single Environment. This need may arise if a customer is looking to
upgrade bandwidth but the current Interconnect does not have sufficient space
available, and the only way to make the customer’s request complete is by
migrating them to a separate Interconnect which still has the space. As this is
a managed service offering to customers these Migrations are entirely seamless
to the customer, but to ensure it goes that we need to follow a few protocols.

A migration is started by a provider issuing a **CreateConnection** call using
the existing Shared UUID of the connection that is being migrated. This is done
to emphasize a “Make Before Break” policy. The “new” **Connection** should be
created following the same pattern as with any **CreateConnection** call, the
only difference being that these connections all funnel back to the same
customer object. When the “Make” path is complete, the customer will
temporarily have double the number of typical connections.

Once the new bring-ups are complete, the next objective is to be able to safely
call **DeleteConnection** on the old **Connection** resource. Remember that
even though we are reusing the same UUID, there are actually still two separate
IDs in the object space by full identifier since each of the Connections is
parented by different interconnects. In order to ensure that no packet loss
occurs while removing the old features, the migration initiator should start by
de-preferencing the routes on those respective features. Failure to do so is
likely to lead to brief packet loss across the old feature links. This should
be the same procedure used when deleting features.

Once it is safe to do so from a routing perspective, the call to
**DeleteConnection** can be made to actually begin the process of removing the
previous components.

## **Maintenance Coordination**

Devices that comprise the same **interconnect** should never be under scheduled
maintenance at the same time, in an effort to minimize the risk of downtime.
This is to be attained by sharing scheduled maintenance events on a per
**Channel** basis, and leveraging that information locally when establishing
new scheduled maintenance events. Note that we recognize that emergency
maintenance does come up, the coordination below does not apply. However, we
still expect that emergency maintenance is presented within the API.

Whenever a new scheduled maintenance arises, the respective Provider will first
inform the other of the change in maintenance status — this is done using the
**NotifyMaintenance** operation on the respective channel. This is a push
style notification, merely indicating a change in the list. Once notified, the
peer is responsible for fetching these events using the
**GetMaintenanceEvents** operation.

Given that we are looking to coordinate across such a large number of routers
overall, it is worth pointing out that a device may actually have multiple
channels on it, each belonging to separate interconnect objects. Care should be
taken to avoid “deadlocking” maintenance due to overlapping interconnects.
Imagine a case where interconnect 1 is placed on the following devices:
\[a,b,c,d\] and interconnect two is placed on \[a,e,c,d\]. If a third
interconnect was placed on \[b,e\], you would no longer be able to perform
maintenance on those routers. As such, we strongly prefer that interconnects
share the same device groupings or are provisioned on non-overlapping routers.
This should be checked at provisioning time.

## **MACsec Key Rotation**

There is an additional object defined in the specification, **MacSecKeys**,
which is a child object of a given **Channel**. The **MacSecKey** object simply
records the **CAK/CKN.** At a known cadence, we will seek to rotate the keys
being used on each individual **Channel**, but never on more than a single
**Channel** at a time. By default, the rotation kickoff burden should be split
50/50 between the providers—this helps to ensure that failures of automation
don’t take out all four channels. Providers can agree to split the burden as
necessary. For example, if a given provider is unable to accept keys inbound
to apply to their router, then they would become effectively responsible for
handling all rotations. It is strongly encouraged that all providers have the
ability to accept external keys—if both providers are unable to accept
external keys , then it will be impossible

To begin a rotation, the partner which is going to be handling the rotation
first starts by installing a new MACsec Key onto their actual device in a
standby manner. Once that is complete, the other partner is informed of the new
key via the **CreateMacSecKey** operation. Upon receipt of the new key, the
receiving partner first has to confirm that a rotation on this channel at this
time is permissible. Assuming it is, the new key is then configured on the
respective channel. Each partner will monitor that the key has become active
from the perspective of the router, and will set the **`active=true`** field
accordingly.

As previously mentioned, there are some rules governing the rotations. In
particular:

*   Only 1 **Channel** within an **Interconnect** can be in rotation at any
    given time.
*   There is a two hour “bake time” after rotating a **Channel** where no other
    rotations can occur.
*   A **Channel** can only be rotated by the provider listed in the
    **`macsecManagerProvider`** field of the **Channel** itself.

If any of the above rules would be violated by a **CreateMacSecKey** operation,
then the operation should be declined with a **`409 Conflict`** response.

Once the new key has become active, it is also the responsibility of the
rotation initiator to delete the old key using the **DeleteMacSecKey**
operation. Any **DeleteMacSecKey** operation must be declined if the target
key is currently considered **Active**. When marking a new key as **Active**,
it is implied that the other keys will all be marked as **Inactive**, thus
permitting the delete to come through. Similar to the create operation,
**DeleteMacSecKey** cannot be sent by anyone other than the listed
**`macsecManagerProvider`** and should be declined outright if received
otherwise.

These same protocols can also be used for the initialization of the first
**MacSecKey** objects on a **Channel**. The only difference is that multiple
**Channels** can be rotated in parallel and the “bake time” merely applies to
future rotations on the same **Channel.** Once this protocol is working, this
is much more streamlined than manually configuring the keys.

As for failure modes, it is generally agreed that **Fail** **Closed** is the
safest option to avoid ever sending unencrypted traffic. Also, because each key
is completely independent of one another, we are often able to simply “fail
forward” and skip directly past a failed key rather than trying to move back.

## **Confused Deputy Resolution \[WIP\]**

In order to ensure that customer impacting actions—specifically
**CreateConnection, UpdateConnection,** and **DeleteConnection**—are the direct
result of actual customer requests, we need to have some way to confirm that
the source of the call was in fact the same user. This process is generally
called Confused Deputy Resolution, and aims to avoid taking actions on the
customer’s behalf without explicit customer actions.

We acknowledge that some **DeleteConnection** operations may come as a
consequence of an unpaid bill, or account termination, etc. We are still
formalizing how to handle such cases automatically, though we expect that
support case (either manual or through the issues APIs) would be needed to
produce the necessary audit events on the peer.

To handle this, each Provider must provide an out-of-band way for the peer
provider to be able to prove that the inciting action came from the same
registered account. For example, AWS handles this process using a call to the
`STS::GetCallerIdentity` function. When the customer makes the request, we use
their authentication session to create a presigned request to the
`STS::GetCallerIdentity` API. Rather than directly invoke that call, we package
up the components of this request in such a way that the partner can
effectively reassemble an entire HTTP request and merely execute it. Since the
request was effectively presigned by the customer, the invocation is effectively
made by that user themselves. This will allow the other provider to obtain the
associated account number or external ID and then compare that to the previously
exchanged IDs. The mutation request should only be accepted if the known ID
and the utilized ID actually match.