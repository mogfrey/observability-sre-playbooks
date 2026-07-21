# Runbook: Tempo object-store timeout

## Example symptom

Tempo fails during startup or storage access with an error similar to:

```text
failed to create store: unexpected error from ListObjects:
Get "https://tempo-traces-demo.s3.eu-west-1.amazonaws.com/?prefix=traces":
dial tcp 10.42.192.15:443: i/o timeout
```

The private IP in this example suggests that DNS resolved the S3 name to a private endpoint. The timeout means name resolution completed, but a TCP/TLS path or endpoint response did not complete in time.

## Do not restart first

Capture the full error, resolved addresses, container network configuration, routes and recent infrastructure changes before restarting Tempo. A restart can temporarily hide intermittent routing or endpoint failures.

## 1. Confirm configuration

Verify without printing credentials:

- bucket name and region;
- object prefix;
- endpoint override, if configured;
- path-style versus virtual-hosted-style addressing;
- workload identity, instance role or credential source;
- proxy and `NO_PROXY` settings;
- timeout and retry configuration.

A region mismatch can redirect requests unexpectedly. An endpoint override can bypass the intended private DNS path.

## 2. Test from the same network namespace

Run diagnostics inside the Tempo pod or container, not only from the host:

```bash
getent ahostsv4 tempo-traces-demo.s3.eu-west-1.amazonaws.com
cat /etc/resolv.conf
ip route
curl -vk --connect-timeout 5 https://tempo-traces-demo.s3.eu-west-1.amazonaws.com/
```

For Kubernetes:

```bash
kubectl -n observability exec deploy/tempo -- getent ahostsv4 tempo-traces-demo.s3.eu-west-1.amazonaws.com
kubectl -n observability exec deploy/tempo -- sh -c 'cat /etc/resolv.conf; ip route'
```

If the image lacks tools, use an approved ephemeral debug container attached to the same pod network namespace.

## 3. Separate DNS, TCP, TLS and authorization

### DNS

- Does the name resolve consistently?
- Is the result expected to be public or private?
- Does split-horizon DNS return the endpoint addresses in the workload VPC/VNet?
- Are private DNS associations or conditional forwarders attached to the correct network?

### TCP

```bash
nc -vz -w 5 <resolved-private-ip> 443
```

A timeout points toward routes, security groups/NSGs, network ACLs, firewalls, endpoint subnets or return routing. A connection refusal means the destination was reached but did not accept the connection.

### TLS

```bash
openssl s_client -connect tempo-traces-demo.s3.eu-west-1.amazonaws.com:443 \
  -servername tempo-traces-demo.s3.eu-west-1.amazonaws.com </dev/null
```

Confirm the SNI name and certificate chain. Do not permanently disable certificate verification to work around a trust or interception problem.

### Authorization

Once TCP/TLS works, an HTTP `403` is useful evidence: the network path is functioning and the investigation should move to identity, endpoint policy, bucket policy, encryption key permissions or request signing.

## 4. Inspect the cloud path

For an AWS interface or gateway endpoint design, review:

```bash
aws ec2 describe-vpc-endpoints --region eu-west-1
aws ec2 describe-route-tables --region eu-west-1
aws ec2 describe-security-groups --region eu-west-1
aws ec2 describe-network-acls --region eu-west-1
```

Validate:

- the workload subnet route table has the intended S3 path;
- the endpoint is in `available` state;
- endpoint security-group ingress allows TCP 443 from the workload source;
- return traffic is permitted;
- NACLs allow the connection and ephemeral return ports;
- endpoint policy permits the required bucket operations;
- the bucket policy does not require a different VPC endpoint;
- transit, firewall or centralized egress routes are symmetric.

Use VPC Flow Logs or equivalent network telemetry to find `REJECT` records for the workload source, endpoint IP and destination port 443.

## 5. Check workload controls

In Kubernetes, review NetworkPolicies and CNI-specific policies. In Podman or Docker, compare host routing with the container bridge and namespace routing. A host-level `curl` can succeed while the application container fails.

```bash
kubectl -n observability get networkpolicy -o yaml
podman inspect <tempo-container>
podman exec <tempo-container> ip route
```

## 6. Validate object-store operations

From the same identity and network path:

```bash
aws s3api get-bucket-location --bucket tempo-traces-demo
aws s3api list-objects-v2 --bucket tempo-traces-demo --prefix traces --max-keys 1
```

Do not test with a personal administrator identity if Tempo uses a workload role. Success with a different identity proves little about the application path.

## Likely remediation categories

- associate the correct route table with the S3 gateway endpoint;
- allow HTTPS from the actual workload subnet or security group to an interface endpoint;
- repair private DNS zone association or forwarding;
- correct asymmetric transit/firewall routing;
- add the required bucket, endpoint or KMS permissions;
- correct bucket region or endpoint configuration;
- include the service endpoint in `NO_PROXY` when an enterprise proxy should not handle private traffic;
- update NetworkPolicy to permit the documented object-store flow.

## Recovery verification

Confirm all of the following:

1. DNS returns the intended addresses.
2. TCP and TLS complete from the Tempo network namespace.
3. `ListObjectsV2`, read and write operations succeed with the workload identity.
4. Tempo starts without repeated storage retries.
5. A test trace is written, queried and retained correctly.
6. No broad temporary firewall or policy exception remains.
