# Negotiate Workflow

One of the CSP's is treated as the "Negotiator" while the other is considered the "Responder".  For our example, we will assume CSP A to be the "Negotitator".

## Negotiation: Interconnect Selection

The first step in the negotiate workflow is to select the appropriate `Interconnect`.  This value is proposed by effectively "creating" the `Connection` object as a resource underneath that proposed `Interconnect`. Remember that as part of the previous Customer workflow, we have already exchanged what the `ConnectionID` will be, which is a UUID in form. Assuming, we are going to use the `Interconnect` named "interconnect_1" f, the request would go as follows:

```
[Sent from CSP A -> CSP B]
POST /v1/providers/CSP_A/environments/env_csp_a_csp_b_1/interconnects/interconnect_1/connections?connectionId=<UUID>
{
    "name": "connectionId",
    "bandwidthMbps": "500",
}
```

If the chosen `Interconnect` is deemed acceptable, this "create" will succeed:
```
201 CREATED
{
    "name": "connectionId",
    "bandwidthMbps": "500",
    "adminStatus": "unknown",
    "provisioningStatus": "unknown"
}
```
If this interconnect is not acceptable, then it should fail the creation with `409 CONFLICT`

In general, the conflict result should only be used for cases in which the selected `Interconnect` cannot fit the required bandwidth.  This should only occur in cases where there is a race between the two sides (i.e. multiple creations at the same time started on separate CSPs).

If there was a `409 CONFLICT` response, the Negotiator should select a different `Interconnect` which is part of the same `Environment` and retry the initial request.

# Negotiate L3 Parameters

After selecting the `Interconnect` each CSP now knows which sets of hardware/ports are to be used when placing the specific customer `Connection`.  With this info, we are able to start the L3 Parameter negotiations.

Each `Interconnect` contains a certain number of `Channels` (typically 4), and the negotiation process for each of these channels is looking to create a `Feature` that contains the L3 Config for a specific `Channel`.

The parameter selection process is designed to be done separately for each of the necessary channels.  The following sample showcases the negotiation workflow for a single channel.

The general Negotiation paradigm is that when an attempt to create a `Feature` occurs, the Responder either Accepts the `Feature` in full or rejects it.

## Step 1: Request FeatureGuidance

The first step in the process is for the Negotiator to ask the Responder for any guidance that should be taken into consideration.  Since the Negotiator must proposed the ENTIRE `Feature` all at once, this is useful for determining the parameters which the other CSP would not know ahead of time.

```
[Sent from CSP A to CSP B]
GET /v1/providers/CSP_A/environments/env_csp_a_csp_b_1/interconnects/interconnect_1/connections/<UUID>/FeatureGuidance

200 OK
{
  "featureGuidance": [
    {
      "featureType": "FEATURE_TYPE_L3_BASE",
      "channelId": "1",
      "l3BaseGuidance": { 
        "asn": [
          {
            "start": 7224,
            "stop": 7224
          }
        ],
        "subnetGuidanceIpv4": [
          {
            "includeSubnet": "169.264.0.0/16"
            "excludeSubnet": [ ],
            "preferredSubnet": "169.254.0.16/30"
          }
        ],
        "subnetGuidanceIpv6": [
          {
            "includeSubnet": "fe80::/64"
            "excludeSubnet": [ ],
            "preferredSubnet": "fe80::16/125"
          }
        ],
        "allowedSubnetSizeIpv4": [  30,  31 ],
        "allowedSubnetSizeIpv6": [ 126, 127 ]
        "vlan": [
          {
            "start": 100,
            "stop": 4000
          }
        ],
        "mtu": [
          {
            "start": 8500,
            "stop": 8500
          }
        ],
        "maxPrefixes": 1000
      }
    }
  ]
}
```

Couple of notes on the above.  

- First, the "l3BaseGuidance" object only exists because the "featureType" is set to "FEATURE_TYPE_L3_BASE".
    - Each different type of feature will have a different set of expected keys.

- This Feature Guidance request is made against the entire `Connection` and must provide feature guidance for each `Channel` on the underlying `Interconnect`
    - The above sample shows one for brevity.

- The "asn", "mtu", and "vlan" sections provide options as a array of ranges.
    - Each of the ranges is considered inclusive.

- The ASN refers to the asn for the remote CSP provider.

- The MTU lists supported values.  The Negotiator should select the smallest value supported by both CSPs.

- The VLAN lists the available VLANs on the respective router.  
    - This list does NOT need to filter out VLANs which are already used by other `Connections` on the same router.

- The subnetGuidance details work as follows:
    - The "includeSubnet" provides a single large prefix that is expected to be used on the specific router.
    - The "excludeSubnet" lists numerous excluded subnets from the initial inclusion.
        - Note: The exclude list does not need to contain the subnets of other `Feature` configurations for the same channel.  It is assumed that the receiving CSP will be able to filter those out on their own.
        - Mostly this is supposed to be used for any case in which ther other CSP would have no idea about specific restrictions on the subnets themselves.
    - The "preferredSubnet" field specifies exactly one prefix which (at least at the moment of the request) would work for the other CSP.
        - This field is not required, but if it is supplied and acceptable by the Negotiator, it should be used.

- The allowed subnet sizes dictate the prefix lengths that the other CSP is willing to use per `Connection`.
    - The Negotiator should select the longest prefix length that both sides support(ie. highest numeric value, smallest subnet size, etc.)

- The max prefixes is a `Connection` level setting informing the other CSP how many prefixes can be received before the BGP session is disabled.
    - Once again, Negotiator should select the smaller of the supported values.

- The API as shown above was used for ALL channels.  This is the default behavior, but if a single channel is desired or only a specific feature type is needed, the GET can be made as such instead:
```
GET /v1/providers/CSP_A/environments/env_csp_a_csp_b_1/interconnects/interconnect_1/connections/<UUID>/FeatureGuidance?channelId=1&featureType=FEATURE_TYPE_L3_BASE
```

## Step 2: Negotiate L3 Parameters

Once the `FeatureGuidance` has been received, we can start to submit actual negotiation attempts for the L3 Configuration.  While the previous `FeatureGuidance` was requested once for the connection on all channels, the L3 Parameter negotiation is intended to be done on a per-channel basis as this is deemed overall to be more flexible than trying to use the same parameters across all channels.

The process of negotiation the `Feature` configuration entails attempting to "create" the `Feature` on the other CSP via a POST command:
```
[Sent from CSP A to CSP B]
POST /v1/providers/CSP_A/environments/env_csp_a_csp_b_1/interconnects/interconnect_1/connections/<UUID/features?featureId=channel1_l3_config
{
  "name": "channel1_l3_config",
  "channelId": "1",
  "featureConfig": {
    "featureType": "FEATURE_TYPE_L3_BASE",
    "l3BaseConfig": {
      "ipSubnetV4": "169.254.0.16/30",
      "subnetSizeV4": 30,
      "ipSubnetV6": "fe80::16/125",
      "subnetSizeV6": 125,
      "mtu": 8500,
      "vlan": 101,
      "sessionAttr": [
        {
          "providerId": "CSP_A",
          "asn": 65000,
          "hostIpv4": "169.254.0.17",
          "hostIpv6": "fe80::17"
        },
        {
          "providerId": "CSP_B",
          "asn": 7224,
          "hostIpv4": "169.254.0.18",
          "hostIpv6": "fe80::18"
        }
      ],
      "md5BgpPassword": "super_secure_password", 
    }
  }
}
```

Assuming that all is well, CSP B would acknowledge the new object with a `201 CREATED`.  This successful response allows both CSPs to consider this Feature ready for provisioning.

If something is not quite right, then CSP B should reject the configuration with `409 Conflict`.  Upon receiving the 409, CSP A should restart this channel's workflow from the `FeatureGuidance` step above, making use of the form which includes query params channelId and featureType.

**Potential Expansion**

The responder may also provide a body to the 409 in an effort to indicate which parameters are not acceptable.  The do this by listing the keys from the featureConfig which cannot be accepted:

```
409 CONFLICT
{
    "rejectKeys": [
        "vlan",
        "sessionAttr/CSP_A/asn"
    ]
}
```
This behavior is entirely optional, but when used would indicate that CSP B would have accepted the `Feature` except for the listed keys.  CSP B's hope is that CSP A will use all of the same parameters except for the ones that were rejected.

That said, CSP A is under no obligation to repropose any of the previously used values, and may elect to go back to `FeatureGuidance` as above.
