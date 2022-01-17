# Terragrunt pattern with Flux & Kustomize 

### Proof-of-concept 

## Overview

This repo emulates the behavior of Terragrunt using Kustomize and the Terraform controller for flux.

One of Terragrunt's strengths is the ability to reduce the repetition required for multi-environment, multi-account, and potentially, multi-region configurations; this repository contains a set of dummy Terraform resources (stacks) provisioned in the manner of a typical Terragrunt project but using Kustomize, Flux and the Terraform controller to achieve the same behavior.

The structure of the repo and function of the various kustomizations is described below.

### Environments

The environments directory provides the entrypoint for flux. Each terraform environment gets a namespace and a flux `Kustomization` targeting the kustomize overlays for that environment.

```
environments/ # <- This is the flux entrypoint
├── flux-system # <- flux controllers, these could live elsewhere
│   ├── gotk-sync.yaml
│   ├── kustomization.yaml
│   └── gotk-components.yaml
├── tf-dev # <- terraform entrypoint for development environment
│   ├── kustomize.yaml # <- Flux Kustomization resource pointing to ./terraform/dev
│   ├── ns.yaml
│   └── kustomization.yaml
├── tf-stg
│   ├── kustomize.yaml # <- Flux Kustomization resource pointing to ./terraform/stg
│   ├── ns.yaml
│   └── kustomization.yaml
├── tf-prd
│   ├── kustomize.yaml # <- Flux Kustomization resource pointing to ./terraform/prd
│   ├── ns.yaml
│   └── kustomization.yaml
└── kustomization.yaml
```

### Stacks

Stacks contains the base resources used to model different logical groups of infrastructure.

Each stack consists of a `GitRepository` and `Terraform` base for which the source repository is patched in the stack-overlay.

Ideally, dependencies between stacks would be expressed at this level, but this functionality is currently difficult to achieve. By way of example, the `tf-aws-eks-stack` overlay contains an dependencies file which illustrates how it might be possible to specify dependencies if the `Terraform` resource supported this api.

```
stacks
├── tf-aws-iam-stack
│   ├── kustomization.yaml
│   └── source.yaml # <- kustomize patch setting the source repository url for the Terraform "stack"
├── tf-aws-eks-stack
│   ├── kustomization.yaml
│   ├── dependencies.yaml # <- [currently non-functional]
│   └── source.yaml
├── base
│   ├── kustomization.yaml
│   ├── terraform.yaml
│   └── repo.yaml
```

### Terraform

The Terraform directory contains the final layer of overlays which will render the full set of manifests for each environment. An env-vars file is present at each layer of the directory tree and these are collected by a kustomize `configMapGenerator` in each account. This mechanism enables scoping variables in manner similar to Terragrunt. This approach can be combined with kustomize patches to override variables and settings at any point in the file-system hierarchy. Currently the `varFrom` field in the Flux Terraform controller spec only supports passing a single `ConfigMap` which necessitates this approach and limits the ability to pass outputs between stacks but future work is planned to address this limitation.

Accounts specify the appropriate set of stacks and use a patch to set the semver tag for the stack in question. This approach enables stacks to be promoted between accounts, regions and environments.

```
terraform
├── globals.env # <- variables available to all accounts, all regions & all environments
├── dev # <- environment
│   ├── environment.env # <- variables available to all stacks in development environment
│   ├── kustomization.yaml
│   ├── eu-west-1 # <- region
│   │   ├── region.env # <- variables for all stacks in this region
│   │   ├── kustomization.yaml # <- kustomization at each layer enables patching
│   │   ├── account-02 # <- account alias
│   │   │   ├── account.env # <- variables for all stacks in this account
│   │   │   ├── kustomization.yaml # <- resources lists stack overlays
│   │   │   └── versions
│   │   │       ├── tf-aws-iam-stack.yaml # <- patches `GitRepository` semver from stack overlay
│   │   │       └── tf-aws-eks-stack.yaml
│   │   └── account-01
│   │       ├── account.env
│   │       ├── kustomization.yaml
│   │       └── versions
│   │           ├── tf-aws-iam-stack.yaml
│   │           └── tf-aws-eks-stack.yaml
│   └── us-east-1
│       ├── region.env
│       ├── kustomization.yaml
│       ├── account-02
│       │   ├── account.env
│       │   ├── kustomization.yaml
│       │   └── versions
│       │       ├── tf-aws-iam-stack.yaml
│       │       └── tf-aws-eks-stack.yaml
│       └── account-01
│           ├── account.env
│           ├── kustomization.yaml
│           └── versions
│               ├── tf-aws-iam-stack.yaml
│               └── tf-aws-eks-stack.yaml
├── stg
│   ├── environment.env
│   ├── kustomization.yaml
│   ├── eu-west-1
│   │   ├── region.env
│   │   ├── kustomization.yaml
│   │   ├── account-02
│   │   │   ├── account.env
│   │   │   ├── kustomization.yaml
│   │   │   └── versions
│   │   │       ├── tf-aws-iam-stack.yaml
│   │   │       └── tf-aws-eks-stack.yaml
│   │   └── account-01
│   │       ├── account.env
│   │       ├── kustomization.yaml
│   │       └── versions
│   │           ├── tf-aws-iam-stack.yaml
│   │           └── tf-aws-eks-stack.yaml
│   └── us-east-1
│       ├── region.env
│       ├── kustomization.yaml
│       ├── account-02
│       │   ├── account.env
│       │   ├── kustomization.yaml
│       │   └── versions
│       │       ├── tf-aws-iam-stack.yaml
│       │       └── tf-aws-eks-stack.yaml
│       └── account-01
│           ├── account.env
│           ├── kustomization.yaml
│           └── versions
│               ├── tf-aws-iam-stack.yaml
│               └── tf-aws-eks-stack.yaml
└── prd
    ├── environment.env
    ├── kustomization.yaml
    ├── eu-west-1
    │   ├── region.env
    │   ├── kustomization.yaml
    │   ├── account-02
    │   │   ├── account.env
    │   │   ├── kustomization.yaml
    │   │   └── versions
    │   │       ├── tf-aws-iam-stack.yaml
    │   │       └── tf-aws-eks-stack.yaml
    │   └── account-01
    │       ├── account.env
    │       ├── kustomization.yaml
    │       └── versions
    │           ├── tf-aws-iam-stack.yaml
    │           └── tf-aws-eks-stack.yaml
    └── us-east-1
        ├── region.env
        ├── kustomization.yaml
        ├── account-02
        │   ├── account.env
        │   ├── kustomization.yaml
        │   └── versions
        │       ├── tf-aws-iam-stack.yaml
        │       └── tf-aws-eks-stack.yaml
        └── account-01
            ├── account.env
            ├── kustomization.yaml
            └── versions
                ├── tf-aws-iam-stack.yaml
                └── tf-aws-eks-stack.yaml
```

### Transforms

The `_transforms` directory contains kustomize `Transformers` that help grease the wheels of the multi-environment setup. In the current setup they extract values from the `ConfigMap` generated per account and use these variables to generate appropriate prefixes for named resources and to configure appropriate labels.
```
_transforms
├── env-labels.yaml
├── region-labels.yaml
├── add-vars-from-config.yaml
├── env-prefix.yaml
├── region-prefix.yaml
├── account-prefix.yaml
├── account-labels.yaml
└── name-from-repo.yaml
```

## Rendering Resources

Render all dev accounts' stacks:
`kustomize build --load-restrictor=LoadRestrictionsNone dev/`

Render staging stacks for account alias `account-01` in the `eu-west-1` region:
`kustomize build --load-restrictor=LoadRestrictionsNone stg/eu-west-1/account-01/`

`./terraform/stg/us-east-1/kustomization.yaml` demonstrates patching a resource to override a variable for a particular region, in this case the number of eks worker nodes:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
patches:
- target:
    kind: ConfigMap
  patch: |-
    - op: replace
      path: /data/eks_node_count
      value: 20
resources:
- account-01/
- account-02/
```

## Questions
- dependencies: can be done with Kustomize but creates difficulties in relation to variable inheritance
- variables: can't do variable inheritance alongside passing outputs between stacks until we `varsFrom` supports multiple inputs.
- auth: how best to configure sts / impersonation per account?

## Deploying

Fork this repo:
```bash
gh repo fork --clone phoban01/flux-terragrunt
cd flux-terragrunt
```

Configure AWS Credentials:
```bash
export AWS_ACCESS_KEY_ID=<insert key id>
export AWS_SECRET_ACCESS_KEY=<insert secret key>
kubectl create secret generic aws-creds -n flux-system \
    --from-literal=aws_access_key_id=$AWS_ACCESS_KEY_ID \
    --from-literal=aws_secret_access_key=$AWS_SECRET_ACCESS_KEY
```

Bootstrap Flux:
```bash
export REPO=$(gh repo view --json name --jq .name)
export OWNER=$(gh repo view --json owner --jq .owner.login)
flux bootstrap github \
    --owner $OWNER \
    --repository $REPO \
    --personal true \
    --path environments
```

Install the Terraform Controller:
```bash
export TF_CON_VER=v0.6.0
kubectl apply -f https://github.com/chanwit/tf-controller/releases/download/${TF_CON_VER}/tf-controller.crds.yaml
kubectl apply -f https://github.com/chanwit/tf-controller/releases/download/${TF_CON_VER}/tf-controller.rbac.yaml
kubectl apply -f https://github.com/chanwit/tf-controller/releases/download/${TF_CON_VER}/tf-controller.deployment.yaml
```

Configure the Terraform Controller to use AWS credentials (hack):
```bash
kubectl patch deploy -n flux-system tf-controller --type=json -p \
'[
    {   "op": "add",
        "path": "/spec/template/spec/containers/0/env/0",
        "value": {
            "name": "AWS_SECRET_ACCESS_KEY",
            "valueFrom": {
                "secretKeyRef": {
                    "name": "aws-creds",
                    "key": "aws_access_key_id"
                }
            }
        }
    },
    {   "op": "add",
        "path": "/spec/template/spec/containers/0/env/0",
        "value": {
            "name": "AWS_SECRET_ACCESS_KEY",
            "valueFrom": {
                "secretKeyRef": {
                    "name": "aws-creds",
                    "key": "aws_secret_access_key"
                }
            }
        }
    },
]'
```

