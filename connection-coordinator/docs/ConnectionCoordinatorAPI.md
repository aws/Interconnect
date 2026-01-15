# Connection Coordinator API

This document is meant to be a supplement to the OpenAPI specification.  Its objective is to provide some clarity around the overall object hierarchy and what each item actually represents.

## Primary Object Types

### Provider
The base type of `provider` is used to delineate the respective CSP from which requests are being made.  For example, as Amazon, we would make requests toward partners using `/providers/aws` as our prefix.  This is used as a way of ensuring resource level permissions and isolation server side.

### Environment
The `environment` object is used to uniquely describe connectivity between two partner CSPs into specific regions.  This is something that the customer would select when creating their connection and would distinctly identify the two regions that are being connected.

For example, a single `environment` could describe the generic path between (AWS us-east-1) and (GCP us-east4) and another could describe a path (AWS us-west-2) and (GCP us-west1).

### Interconnect
The `interconnect` object describes a bundle of `channels` upon which the CSP would place customer `connections`.  Each `interconnect` exists as part of an `environment` and each `environment` may have many interconnects.  Customers never see the `interconnect` objects, they are solely used by the CSPs.

The `interconnect` objects reflect the pre-cabled capacity that will be sold to customers and reserved for their use.

### Channel
The `channel` object is a single LAG that resides as part of an `interconnect`.  There are 4 `channels` per `interconnect`, and each one is a LAG comprising the same number of 100G (or 400G, but must be uniform) ports, resulting in identical capacity on all `channels`.

### Connection
The `connection` object represents an explicit customer request for a specific bandwidth across a specific `environment`.  After a customer requests a `connection`, our APIs will be used to negotiate the specific `interconnect` to place this `connection` on.  The customer has no influence over the `interconnect` choice.

### Feature
The `feature` object is an extensible object to describe anything that we may want to support on a given connection.  At this time, there is only one feature type, which is Channel Config.

The Channel Config `feature` is used to convey a negotiated set of L3 parameters including VLAN, IPv4 Subnet, IPv6 Subnet, and MTU.  Each of these parameters is negotiated as part of the customer's creation.  Once again, the customer has no real influence over the chosen parameters as they are distinctly negotiated between the two CSPs.

When this Channel config is agreed upon, a hosted connection of the appropriate bandwidth is created on the LAG represented by the `channel`.

## Connection Coordination Milestones

When a customer is looking to create a `Connection` there are 4 major milestones we must achieve:

1. Establish Customer Intent
2. Negotiate Relevant Features Parameters
3. Provision Relevant Features
4. Activate Connection

## Establish Customer Intent

To establish this intent, we require that a given customer informs both parties of their desire to create a connection.  The standard form of this involves the customer invoking a `CreateConnection` operation on one CSP, and then bringing metadata to the other CSP to perform a pairing `AcceptConnection` operation.

### Create Connection at CSP A

The customer needs to make a call to the appropriate API on CSP A, specifically the API which performs the function of `CreateConnection`.  Worth noting that this functionality is required at each CSP, but the specification does not set out any specific API names of constructs that must be used.

Overall, this `CreateConnection` operation needs to determine at least the following information:
- Remote CSP Account ID
- Environment
- Bandwidth

#### Remote CSP Account ID

When creating the connection, the customer must supply a means of identifying which account on the other CSP they intend to connect.  In the case of AWS, this would be the account number, and in the case of GCP, this would be the Project Name.  This is used to ensure that only a single destination account can accept the details of this specific connection when it is time for the later confirmation step.

#### Environment Choice

The `Environment` dictates which CSPs and regions the resulting connection will span.  The specific `Environment` choice may also represent the selection of a specific additional quality in the underlying connection--no such additional qualities exist at this time, but this is the scope at which the distinction would be made.

#### Bandwidth Choice

Once the specific `Environment` is known, the user can also select their desired bandwidth.  Bandwidth is offered at discrete tiers, and each specific `Environment` may offer a different set of selections.  The permissible set of bandwidths is laid out within the specification itself.

#### Activation Key

The net result of a call to this API is to create a pending `Connection` object in CSP A and to return to the customer an `ActivationKey`.  This `ActivationKey` contains details about the underlying connection parameters set out by the customer in this call.

Note that to this point, no calls have been made from CSP A to CSP B--that won't happen until after the customer gives CSP B the `ActivationKey`

### Accept Connection at CSP B

With the `ActivationKey` from CSP A in hand, the customer needs to go login to CSP B.  Here they will call the appropriate API serving the function of `AcceptConnection`.  This API needs to take in the `ActivationKey`.  CSP B should read the details out of the `ActivationKey` and provide that information to the customer so they can confirm that the information looks correct.

Assuming the customer accepts the details, the `Connection` object on CSP B can be created in a pending state--this object is tentative pending successful confirmation.

### Confirming the Activation Key

Notice that in the above step, CSP B did not yet ask CSP A if the `ActivationKey` was real.  While we may have already created a stand-in customer object on CSP B, we must still ask CSP A to confirm that the provided `ActivationKey` is in fact real.

This is done using the `ConfirmActivationKey` API scoped to the desired environment.  If CSP A responds to this request with success, then we can claim that we have completely established the customer's intent.

## Negotiate Relevant Features Parameters

For the purposes of a negotiation phase, one CSP is considered the "Negotiator" while the other is considered the "Responder".  The Negotiator will make proposals to fulfill specific needs, and the Responder either accepts or rejects the proposals in full.

### Negotiation: Interconnect Selection

The first step in the negotiate workflow is to select the appropriate `Interconnect`.  This value is proposed by effectively "creating" the `Connection` object as a resource underneath that proposed `Interconnect`. If this interconnect is not acceptable, then it should fail the creation with `409 CONFLICT`.

In general, the conflict result should only be used for cases in which the selected `Interconnect` cannot fit the required bandwidth.  This should only occur in cases where there is a race between the two sides (i.e. multiple creations at the same time started on separate CSPs).

If there was a `409 CONFLICT` response, the Negotiator should select a different `Interconnect` which is part of the same `Environment` and retry the initial request.

### Negotiate L3 Parameters

After selecting the `Interconnect` each CSP now knows which sets of hardware/ports are to be used when placing the specific customer `Connection`.  With this info, we are able to start the L3 Parameter negotiations.

Each `Interconnect` contains a certain number of `Channels` (optimally 4), and the negotiation process for each of these channels is looking to create a `Feature` that contains the L3 Config for a specific `Channel`.

The parameter selection process is designed to be done separately for each of the necessary channels.  The following sample showcases the negotiation workflow for a single channel.

The general Negotiation paradigm is that when an attempt to create a `Feature` occurs, the Responder either Accepts the `Feature` in full or rejects it.

#### Step 1: Request FeatureGuidance

The first step in the process is for the Negotiator to ask the Responder for any guidance that should be taken into consideration.  Since the Negotiator must proposed the ENTIRE `Feature` all at once, this is useful for determining the parameters which the other CSP would not know ahead of time.  Note, the guidance itself is requested on the `Connection` object, and the response will provide guidance for each desired `Feature` independently.

In general, the guidance includes details for each `Feature` such as:
- ASN of the remote network
- MTU
- IPv4/IPv6 details

The IPv4 and IPv6 details provide narrowing semantics around the BGP peering parameters that will be negotiated, specifically a large subnet of "acceptable" space, with smaller subnets of exclusions.


#### Step 2: Negotiate L3 Parameters

Once the `FeatureGuidance` has been received, we can start to submit actual negotiation attempts for the L3 Configuration.  While the previous `FeatureGuidance` was requested once for the connection on all channels, the L3 Parameter negotiation is intended to be done on a per-channel basis as this is deemed overall to be more flexible than trying to use the same parameters across all channels.

The process of creating a `Feature` involves quite literally creating a `Feature` object on the corresponding `Connection`, using the `CreateFeature` operation.  A successful creation will elicit a 200 response.

If something is not quite right, then the responder should reject the configuration with `409 Conflict`.  Upon receiving the 409, the negotiator should restart this channel's workflow from the `FeatureGuidance` step above, making use of the form which includes query params channelId and featureType.

Once all desired features have been created, we can move onto provisioning the actual hardware.

## Provision Relevant Features

With all Features in place and agreed upon, each CSP now looks to handle the necessary provisioning on their respective internal systems.  This operation is not necessarily guaranteed to be quick, and CSPs should be prepared as such.

After local provisioning is complete, each CSP can independently move onto the final step, Activate Connection.

## Activate Connection

Once all provisioning is done, each CSP can indicate that the connection has become active by setting the `VerificationState` accordingly.  Additionally, they must call `NotifyConnectionStatus` API to inform the other CSP of the status change.

From here, all customer routes should be exchanged and connectivity established.