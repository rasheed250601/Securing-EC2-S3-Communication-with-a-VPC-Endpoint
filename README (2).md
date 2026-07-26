# Securing S3 Access with a VPC Gateway Endpoint

Moving S3 access off the public internet path and onto AWS's private backbone — then proving it with a real bucket policy that denies every request unless it arrives through the endpoint, and a live terminal test that fails before the fix and succeeds after it.

**Author:** Mohammed Abdul Rasheed — [linkedin.com/in/mohammed-abdul-rasheed](https://linkedin.com/in/mohammed-abdul-rasheed)

`AWS` · `VPC` · `S3` · `Gateway Endpoint` · `IAM` · `Bucket Policy`

---

## 📌 Project Overview

This project shows why traffic between an EC2 instance and an S3 bucket doesn't have to leave AWS's network at all.

**Starting point:** a VPC with a working Internet Gateway, where an EC2 instance could already reach S3 **over the public internet** using its IAM credentials.

**Goal:** close that public path entirely and force all S3 traffic through a **VPC Gateway Endpoint** instead — then prove, with a real bucket policy and a live terminal test, that access only works when it arrives through the endpoint.

### Why this matters

Without an endpoint, traffic from a private EC2 instance to S3 either needs a public IP + Internet Gateway/NAT, or it silently routes out over the public AWS edge. A Gateway Endpoint keeps that traffic on Amazon's internal network, removes the need for internet egress just to reach S3, and lets a bucket policy enforce "only reachable through this endpoint" as a hard security boundary — not just an IAM permission.

- ✅ Free — no hourly charge for Gateway Endpoints
- ✅ No NAT Gateway required
- ✅ Enforced at the bucket policy layer

---

## 🏗️ Architecture

```
                 INTERNET
                     │
                     ✕  (public path denied)
                     │
   ┌─────────────────────────────────────┐
   │  VPC: Myvpcs3 (10.0.0.0/16)          │
   │                                       │
   │  ┌───────────────┐   ┌─────────────┐│      ┌───────────┐
   │  │  mys3instance  │──▶│  Gateway     ││─────▶│  Amazon S3│
   │  │ Public+Private │   │  Endpoint    ││      │test-s3-   │
   │  │    subnet      │   │ my-s3-endpoint│      │ rasheed   │
   │  └───────────────┘   └─────────────┘│      └───────────┘
   │                                       │
   └─────────────────────────────────────┘
          private AWS network path — never touches the internet
```

## 🧩 Components

| Component | Name | Configuration |
|---|---|---|
| VPC | Myvpcs3 | `10.0.0.0/16` |
| Subnet | my-s3-subnet | `10.0.1.0/24 (us-east-1a)` |
| Route Table | my-s3-route-table | `0.0.0.0/0 → IGW`, `10.0.0.0/16 → local`, `S3 prefix list → Endpoint` |
| VPC Endpoint | my-s3-endpoint | Gateway type, service `com.amazonaws.us-east-1.s3` |
| EC2 Instance | mys3instance | Public `44.197.186.97` / Private `10.0.1.160` |
| S3 Bucket | test-s3-rasheed | Bucket policy denies access outside the endpoint |
| IAM User | tests3 | Configured on the instance via `aws configure` |

---

## 🚀 Step-by-Step Walkthrough

### 1. Baseline — S3 Reachable Over the Internet

**Step 01 — Confirm the VPC and instance**
The starting point was a working VPC (**Myvpcs3**, `10.0.0.0/16`) with an Internet Gateway attached, and an EC2 instance (**mys3instance**) launched with public IP `44.197.186.97` and private IP `10.0.1.160`, sitting in `my-s3-subnet`.

**Step 02 — Launch the instance inside the S3 subnet**
`mys3instance` was launched as a `t3.micro` in `my-s3-subnet`, picking up private IP `10.0.1.160` and public IP `44.197.186.97` — reachable over the internet and, at this point, still routing to S3 the same way: out through the Internet Gateway.

### 2. Lock the Bucket to the Endpoint

**Step 03 — Attach a deny-by-default bucket policy**
A bucket policy was attached to `test-s3-rasheed` that denies every S3 action on the bucket and its objects unless the request's `aws:sourceVpce` matches the endpoint ID. At this stage the policy was written referencing the endpoint before it was fully wired up, so any access — including from inside the VPC — was rejected.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::test-s3-rasheed",
        "arn:aws:s3:::test-s3-rasheed/*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:sourceVpce": "vpce-0695bdb8a858114a8"
        }
      }
    }
  ]
}
```

**Step 04 — Test from the instance: access denied**
Back on `mys3instance`, running `aws s3 ls s3://test-s3-rasheed` after the policy was attached returned **AccessDenied** — an explicit deny in a resource-based policy — because traffic was still leaving through the Internet Gateway, not the (not-yet-connected) endpoint. This confirms the policy is doing real enforcement, not just sitting there unused.

```
$ aws s3 ls s3://test-s3-rasheed
An error occurred (AccessDenied) when calling the ListObjectsV2 operation:
User: arn:aws:iam::550597787354:user/tests3 is not authorized to perform:
s3:ListBucket on resource: "arn:aws:s3:::test-s3-rasheed" because
an explicit deny in a resource-based policy
```

### 3. Create the Gateway Endpoint & Wire the Route

**Step 05 — Create the S3 Gateway Endpoint**
A VPC Endpoint named `my-s3-endpoint` was created for service `com.amazonaws.us-east-1.s3`, type **Gateway**, attached to **Myvpcs3**. Unlike an Interface Endpoint, a Gateway Endpoint doesn't get an IP — it works by inserting a special route into whichever route tables it's associated with.

**Step 06 — Associate the endpoint with the route table**
Associating `my-s3-endpoint` with `my-s3-route-table` added a new route: destination is the S3 prefix list (`pl-63a5400a`), target is the endpoint itself. Now any traffic to S3's IP ranges from this subnet is routed to the endpoint instead of out through the Internet Gateway — the private path now exists alongside the public one.

| Destination | Target | Status |
|---|---|---|
| `pl-63a5400a` | `vpce-0695bdb8a858114a8` | ✅ Active |
| `0.0.0.0/0` | `igw-0e4dc758c9e3de757` | ✅ Active |
| `10.0.0.0/16` | `local` | ✅ Active |

### 4. Validate — Access Restored Through the Endpoint

**Step 07 — Re-run the CLI test: it works**
With the route in place, running `aws s3 ls s3://test-s3-rasheed` again from `mys3instance` now lists the bucket's objects successfully — traffic is leaving through the Gateway Endpoint, matching the `aws:sourceVpce` condition in the bucket policy, and the deny no longer applies.

```
$ aws s3 ls s3://test-s3-rasheed
2026-07-21 05:56:51     455154 1.jpg
```

**Step 08 — Final bucket policy**
The bucket's final state: a `Deny` statement on `s3:*` for the bucket and all objects in it, scoped by a `StringNotEquals` condition on `aws:sourceVpce`. Any request that doesn't arrive through `vpce-0695bdb8a858114a8` is rejected outright, regardless of IAM permissions.

---

## ✅ Conclusion & Key Takeaways

Successfully closed off S3's public-internet path and proved that a private EC2 instance can only reach the bucket through a VPC Gateway Endpoint — enforced independently by routing and by the bucket policy itself, and validated with real, before-and-after CLI tests rather than just inspecting console screens.

| Takeaway | Details |
|---|---|
| **Traffic Never Leaves AWS** | Once the Gateway Endpoint route is in place, S3 traffic from the private subnet travels entirely over Amazon's internal network instead of the public internet. |
| **Policy-Enforced, Not Just Routed** | The bucket policy's `aws:sourceVpce` condition makes the endpoint mandatory — even valid IAM credentials fail if the request doesn't arrive through it. |
| **No NAT Gateway Needed** | A Gateway Endpoint is free and needs no NAT Gateway or extra public IPs just to let private resources reach S3. |
| **Validated, Not Assumed** | The design was proven end-to-end with a real CLI test: denied before the endpoint route existed, working after — not just inspected in the console. |

---

## 🔒 Security Note

The AWS Access Key ID and Secret Access Key visible in the original terminal screenshots have been redacted before publishing. Replace all resource IDs, ARNs, and IPs above with your own before reusing this setup.

---

## 📎 Resources

- [AWS VPC Endpoints for S3 — Documentation](https://docs.aws.amazon.com/AmazonS3/latest/userguide/privatelink-interface-endpoints.html)
- Connect: [linkedin.com/in/mohammed-abdul-rasheed](https://linkedin.com/in/mohammed-abdul-rasheed)
