# container-platform-terraform-cert-manager

This module deploys cert-manager on an EKS cluster with AWS IAM integration using [EKS Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html).

## Overview

cert-manager is a Kubernetes add-on that automates the management and issuance of TLS certificates. This module:

- Deploys cert-manager using the official Jetstack Helm chart (v1.20.3)
- Enables Gateway API support including ListenerSet integration (alpha feature)
- Configures AWS IAM permissions via [EKS Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html) for Route53 DNS challenge validation
- Creates three ClusterIssuers:
  - `letsencrypt-staging` - Let's Encrypt staging environment for testing
  - `letsencrypt-production` - Let's Encrypt production environment
  - `selfsigned` - Self-signed certificate issuer

## Provider Requirements

| Name | Version |
|------|---------|
| terraform | ~> 1.0 |
| aws | ~> 6.0 |
| kubernetes | > 2.0 |
| helm | > 2.0 |
| kubectl | ~> 2.0 |

## Resources Created

- **Kubernetes Namespace**: `cert-manager` namespace with privileged pod security policy
- **Helm Release**: cert-manager v1.20.3
- **EKS Pod Identity**: IAM role with Route53 permissions for DNS-01 challenges
- **ClusterIssuers**: Three pre-configured issuers (staging, production, self-signed)

## Inputs

| Name | Description | Type | Required | Default |
|------|-------------|------|:--------:|---------|
| cluster_name | The name of the EKS cluster | `string` | yes | n/a |
| hostzones | List of Route53 Hosted Zone ARNs that cert-manager is allowed to manage for ACME DNS challenges. Format: `arn:aws:route53:::hostedzone/<ZONE_ID>`. Use `["arn:aws:route53:::hostedzone/*"]` to allow all hosted zones. | `list(string)` | yes | n/a |
| certman_replicas | Number of replicas for cert-manager deployment | `number` | no | `1` |
| webhook_replicas | Number of replicas for cert-manager webhook deployment | `number` | no | `1` |
| cainjector_replicas | Number of replicas for cert-manager cainjector deployment | `number` | no | `1` |

## Outputs

This module does not export any outputs.

## Usage

### Basic Usage

```hcl
module "cert_manager" {
  source = "./modules/cert-manager"

  cluster_name = "my-eks-cluster"
  hostzones    = ["arn:aws:route53:::hostedzone/Z1234567890ABC"]
}
```

### All Hosted Zones

```hcl
module "cert_manager" {
  source = "./modules/cert-manager"

  cluster_name = "my-eks-cluster"
  hostzones    = ["arn:aws:route53:::hostedzone/*"]
}
```

### High Availability

```hcl
module "cert_manager" {
  source = "./modules/cert-manager"

  cluster_name        = "my-eks-cluster"
  hostzones           = ["arn:aws:route53:::hostedzone/*"]
  certman_replicas    = 3
  webhook_replicas    = 3
  cainjector_replicas = 3
}
```

## Using the ClusterIssuers

After deploying this module, you can create certificates using the ClusterIssuers:

### Production Certificate

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com-tls
  namespace: default
spec:
  secretName: example-com-tls
  issuerRef:
    name: letsencrypt-production
    kind: ClusterIssuer
  dnsNames:
    - example.com
    - www.example.com
```

### Staging Certificate (for testing)

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com-tls
  namespace: default
spec:
  secretName: example-com-tls
  issuerRef:
    name: letsencrypt-staging
    kind: ClusterIssuer
  dnsNames:
    - example.com
```

## IAM Permissions

The module automatically configures the following AWS IAM permissions for cert-manager via the [EKS Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html) module:

- `route53:GetChange` (for all change resources)
- `route53:ChangeResourceRecordSets` (scoped to specified hosted zones)
- `route53:ListResourceRecordSets` (scoped to specified hosted zones)
- `route53:ListHostedZonesByName` (global resource)

For the complete IAM policy definition, see the [terraform-aws-modules source code](https://github.com/terraform-aws-modules/terraform-aws-eks-pod-identity/blob/master/cert_manager.tf).

## Notes

- The cert-manager service account is automatically associated with the IAM role via [EKS Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)
- DNS-01 challenge is used for certificate validation, requiring Route53 access
- Let's Encrypt has rate limits; use staging for testing
- Self-signed issuer is available for development/testing purposes
- **Gateway API ListenerSet Support**: This module enables the [`ListenerSets` feature gate](https://cert-manager.io/docs/usage/gateway/#listenerset), which is currently in **alpha state**. This allows cert-manager to automatically provision certificates for Gateway API ListenerSet resources. The feature is still under development and may change in future releases.

## References

- [cert-manager Documentation](https://cert-manager.io/docs/)
- [cert-manager Gateway API Support](https://cert-manager.io/docs/usage/gateway/)
- [cert-manager ListenerSet Support](https://cert-manager.io/docs/usage/gateway/#listenerset)
- [Let's Encrypt Rate Limits](https://letsencrypt.org/docs/rate-limits/)
- [EKS Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)
