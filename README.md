# RFC: Sandbox Egress Traffic Policy 

> **Status:** INPROGESS

## Authors
- Agents Team

## Problem Statement

When users create AI agent sandboxes, those agents often need to interact with external APIs (like GitHub, Stripe, or OpenAI). However, allowing a sandbox completely open access to the internet (`0.0.0.0/0`) is a significant security risk, as a compromised agent could download malware or exfiltrate data. 

We need a simple API that lets customers choose exactly which websites their sandbox is allowed to visit. To make this easy to use, customers must be able to type normal website names (like `api.github.com`) in addition to raw IP addresses.

To achieve this, we will only provide a mechanism for users to add IPs, IP ranges, or DNS names to the allow list. We will not worry about whether a DNS name resolves or whether an IP is valid/reachable at configuration time.

## Goals

- **G1. Block Everything By Default:** The API will automatically block all internet access for a sandbox unless the user specifically provides an allow list.
- **G2. Blindly Accept Names:** Our API will accept whatever website names or IP addresses the user types in, and save them directly to the database without trying to verify if they are real or reachable.
- **G3. Smart Network Enforcement:** We will rely on the cluster's network layer (like Cilium) to do the actual work of looking up the website names and blocking/allowing traffic while the sandbox is running.
- **G4. Continuous Enforcement:** We will save the exact list the user provided in the database, so the system can constantly make sure the firewall rules stay updated.

## Non-Goals

- **No Website Validation:** Our backend API (`aiagent-service`) will *not* try to ping or look up the website names the user provides to verify if they are real.

## API Surface

The `CreateSandboxRequest` will be extended with an optional `egress` object. 

### Schema Definition
```json
"egress": {
  "mode": "deny_all | allow_list",
  "allow": [
    {
      "host": "string (IP, CIDR, or DNS name)",
      "protocol": "TCP | UDP",
      "ports": [int]
    }
  ]
}
```

## Architecture & Implementation

### 1. Database Persistence
The accepted `egress` block is serialized to JSON and stored in a new `egress_policy JSONB` column in the `sandboxes` table. 

### 2. Creating the Firewall Policy
Since standard Kubernetes firewalls don't understand website names, our backend will generate a special `CiliumNetworkPolicy` instead. 
- It will take the exact website name the user typed and place it into a special `toFQDNs` field.
- This creates a strict security rule that only applies to that specific user's sandbox.

### 3. How it Works (The Magic)
When the AI agent tries to visit an allowed website (e.g., `api.github.com`), this is exactly what happens behind the scenes:
1. The agent asks the network, "What is the IP address for api.github.com?"
2. Cilium secretly listens to this question, intercepts the answer, and instantly updates the firewall to allow traffic to that specific IP address.

## Validation Rules

- **Reject Overlapping/Contradictory Rules:** We will reject the request if the user provides duplicate rules or IP ranges that overlap. This keeps the firewall rules clean.
- **Reject Open Internet:** The API will instantly reject any rule specifying `0.0.0.0/0` (or `::/0`), as this grants unrestricted internet access. The only exception is if an internal platform engineer bypasses this by setting an explicit, audit-logged `allow_internet: true` flag for special debugging cases.
