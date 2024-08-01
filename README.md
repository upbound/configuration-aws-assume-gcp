# Configuring Workload identity federation between GCP and AWS EKS

Workload identity federation is the process of impersonating an identity in one cloud provider from the other without long lived keys.
In this configuration-aws-assume-gcp we will walk through how to setup federation between a workload (provider-gcp-compute) running on AWS EKS to GCP.

Here are the steps we need to complete on the GCP side as a prerequisite.
We'll also walk you through the customizations needed for provider-gcp-compute so it can use the impersonated service account on the GCP side.

# GCP
## Enable Additional GCP Services
The Security Token Service API and the IAM Service Account Credentials API need to be enabled in our GCP project.

```bash
gcloud services enable sts.googleapis.com
gcloud services enable iamcredentials.googleapis.com
```

## Create GCP Service Account

Now we need to create the GCP Service Account which provider-gcp-compute will authenticate with in order to Create,Update,Delete GCP resources.

```bash
gcloud iam service-accounts create configuration-aws-assume-gcp \
    --description="configuration-aws-assume-gcp for providers which runs on AWS." \
    --display-name="configuration-aws-assume-gcp"
```

We need to ensure that the new Service Account has the appropriate permission to Create,Update,Delete resources in the desired project in GCP.

```bash
gcloud projects add-iam-policy-binding crossplane-playground \
--member="serviceAccount:configuration-aws-assume-gcp@crossplane-playground.iam.gserviceaccount.com" \
--role="roles/compute.admin"
```

## Create GCP Workload Identity Pool

Next, we need to create the GCP Workload Identity Pool.

```bash
gcloud iam workload-identity-pools create configuration-aws-assume-gcp \
    --description="Workload Identity Pool for Providers on AWS." \
    --display-name="Providers AWS Pool" \
    --location="global"
```

## Create Workload Identity Provider

After creating the GCP Workload Identity Pool we need to create the Workload Identity Provider.
This example does not cover how to harden the Identity Provider, which can be achieved by passing the `--attribute-condition` flag.

```bash
gcloud iam workload-identity-pools \
    providers create-aws configuration-aws-assume-gcp  \
    --account-id=<AWS-ACCOUNT-ID> \
    --description="Workload Identity Provider for Providers on AWS." \
    --display-name="Providers AWS" \
    --location="global" \
    --workload-identity-pool="configuration-aws-assume-gcp"
```

## Download Client library

The Client library config needs to be downloaded, this can be achieved by navigating to the Workload Identity Pool and selecting the CONNECTED SERVICE ACCOUNTS Tab on the right side. The Client library config can be obtained from there. The file does NOT contain sensitive information.

## Bind GCP Service Account Impersonation.

Now, we will create an IAM policy binding.
The below command allows all identites from our pool to impersonate the GCP Service Account we created earlier by binding the IAM role `roles/iam.workloadIdentityUser` to the GCP service account.

This example does not cover how to harden with granular scoping for service account impersonation.

```bash
gcloud iam service-accounts add-iam-policy-binding \
    configuration-aws-assume-gcp@crossplane-playground.iam.gserviceaccount.com \
    --member "principalSet://iam.googleapis.com/projects/<GCP Project Number>/locations/global/workloadIdentityPools/configuration-aws-assume-gcp/*" \
    --role "roles/iam.workloadIdentityUser"
```

# AWS

On the AWS side, you'll need an EKS cluster with an OIDC provider enabled to use IRSA, or you can use the newer PodIdentity option.
This allows you to assign an IAM role to a Kubernetes service account for role assumption.
We won't go into detailed setup instructions here, as there are various configurations available for implementing this.

## Create configmap with GCP Client library

Earlier, we downloaded the GCP Client library for the Workload Pool.
Now, we need to create a ConfigMap using its content.
Just to note, this file does NOT contain any sensitive information.

```bash
kubectl -n upbound-system create configmap gcp-default-credentials --from-file=gcp_default_credentials.json
```

## Create DeploymentRuntimeConfig

We need to create a DeploymentRuntimeConfig to customize the name of the service account and add the required IRSA annotation for assuming an AWS role. Additionally, we need to configure provider-gcp-compute to locate the client library JSON file, which will be used later for impersonating a GCP service account.

```yaml
apiVersion: pkg.crossplane.io/v1beta1
kind: DeploymentRuntimeConfig
metadata:
  annotations:
  name: upbound-provider-gcp-compute
spec:
  serviceAccountTemplate:
    metadata:
      annotations:
        eks.amazonaws.com/role-arn: arn:aws:iam::<AWS-ACCOUNT-ID>:role/configuration-aws-assume-gcp-f4drp
      name: upbound-provider-gcp-compute
  deploymentTemplate:
    spec:
      replicas: 1
      selector: {}
      template:
        metadata:
          labels: {}
        spec:
          containers:
          - env:
            - name: GOOGLE_APPLICATION_CREDENTIALS
              value: /tmp/gcp_default_credentials.json
            name: package-runtime
            volumeMounts:
            - mountPath: /tmp/
              name: gcp
          volumes:
          - configMap:
              items:
              - key: gcp_default_credentials.json
                path: gcp_default_credentials.json
              name: gcp-default-credentials
            name: gcp
```

## Create ProviderConfig for provider-gcp

We need to create a ProviderConfig to allow impersonation of the service account in GCP.

```yaml
apiVersion: gcp.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    impersonateServiceAccount:
      name: configuration-aws-assume-gcp@crossplane-playground.iam.gserviceaccount.com
    source: ImpersonateServiceAccount
  projectID: crossplane-playground
```