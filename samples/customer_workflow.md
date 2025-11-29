# Customer Workflow Example

## Create Connection at CSP A

The customer needs to make a call to the appropriate API on CSP A, specifically the API which performs the function of `CreateConnection`.  Worth noting that this functionality is required at each CSP, but the specification does not set out any specific API names of constructs that must be used.

Overall, this `CreateConnection` operation needs to determine at least the following information:
- MultiCloud Environment
- Desired Bandwidth

The Environment dictates which CSP and region the connection should connect from and to.  The desired bandwidth is given, and is selected from an agreed upon list of options between the two CSPs.

The net result of a call to this API is to create a pending `Connection` object in CSP A and to return to the customer an `ActivationKey`.  This `ActivationKey` contains details about the underlying connection parameters set out by the customer in this call.

Note that to this point, no calls have been made from CSP A to CSP B--that won't happen until after the customer gives CSP B the `ActivationKey`

## Accept Connection at CSP B
With the `ActivationKey` from CSP A in hand, the customer needs to go to CSP B.  Here they will call the appropriate API serving the function of `AcceptConnection`.  This API needs to take in the `ActivationKey`.  CSP B should read the details out of the `ActivationKey` and provide that information to the customer so they can confirm that the information looks correct.

Assuming the customer accepts the details, the `Connection` object on CSP B can be created in a pending state--this object is tentative pending successful confirmation.  

At this point is the first time that we leverage the symmetric API:

```
[Sent by CSP B to CSP A]
POST ​/v1​/providers​/CSP_B/environments​/env_csp_a_csp_b_1​/ConfirmActivationKey
{
    "key": "<ACTIVATION_KEY_CONTENT>"
}
```

Note: ActivationKey is just a Base64 encoded JSON object:

```
ActivationKey := Base64({
    "environment": "env_csp_a_csp_b_1",
    "connectionSizeMbps": "500",
    "keyMaterial": "<Unique Data>"
})
```

CSP A will receive this request, and will attempt to locate the specific `ActivationKey` locally.  If found and valid, then we can move the `Connection` object on CSP A into the next phase, `Negotiation`.  We also need to inform CSP B that the `ActivationKey` is real and that the customer's `Connection` is now ready for negotiations.

This will lead to a response indicating if the key is real or not:
```
200 OK
{
    "keyValid": true
}
```

When receiving this response, if `keyValid=true` then we need to progress the CSP B side of the `Connection` into the ready for negotiations phase.  If `keyValid=false` then we need to delete the `Connection` on CSP B as this was an invalid `ActivationKey` (Spoofed, Tweaked, etc.).

Assuming the successful path was followed, the corresponding `Connection` objects on CSP A and CSP B are now both ready to move to the negotiaton phase.


