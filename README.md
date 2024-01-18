# aws-secrets-sync-with-k8s
Syncing AWS Secrets Manager secrets to Kubernetes secrets in EKS


## Background
AWS Documentation provides instructions and tutorial for setting up ASCP, SecretsManager secret and testing it with a pod - https://docs.aws.amazon.com/secretsmanager/latest/userguide/integrating_csi_driver.html

## Problem
Overall, there are 2 problems in the documentation for Installing and Configuring ASCP:
1. `--set syncSecret.enabled=true` option is not mentioned
2. There is no clear example/documentation on using Secrets provider with ENV variables in a pod. Example is only with mounting secrets as files.


# Deploy ASCP the right way
### with `--set syncSecret.enabled=true` option

```
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install -n kube-system csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --set syncSecret.enabled=true

helm repo add aws-secrets-manager https://aws.github.io/secrets-store-csi-driver-provider-aws
helm install -n kube-system secrets-provider-aws aws-secrets-manager/secrets-store-csi-driver-provider-aws
```


# Customized Tutorial
Prerequisites followed in this tutorial: `eksctl, helm, EKS cluster`
```
brew install eksctl
git clone git@github.com:312-bc/312-eks-common.git
cd 312-eks-common/
make cluster version=1.25
brew install helm
```

Integration: AWS SecretsManager + K8s Secrets
- https://docs.aws.amazon.com/secretsmanager/latest/userguide/integrating_csi_driver.html


**1. Follow above tutorial but with `--set syncSecret.enabled=true` option**
```
helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install -n kube-system csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --set syncSecret.enabled=true

helm repo add aws-secrets-manager https://aws.github.io/secrets-store-csi-driver-provider-aws
helm install -n kube-system secrets-provider-aws aws-secrets-manager/secrets-store-csi-driver-provider-aws
```

**2. Create variables locally** (adjust values as necessary)
```
REGION=us-east-1
CLUSTERNAME=eks-dev
```

**3. Create secret** (adjust values if necessary)
```
aws --region "$REGION" secretsmanager  create-secret --name exchange-app-db-secrets --secret-string '{"dbuser":"myadmin", "dbpassword":"superpass"}'
```

**4. Make sure to replace below secret arn with your current secret ARN, and then Create iam policy that provides access to above secret.** 
```
POLICY_ARN=$(aws --region "$REGION" --query Policy.Arn --output text iam create-policy --policy-name nginx-deployment-policy --policy-document '{
    "Version": "2012-10-17",
    "Statement": [ {
        "Effect": "Allow",
        "Action": ["secretsmanager:GetSecretValue", "secretsmanager:DescribeSecret"],
        "Resource": ["arn:aws:secretsmanager:us-east-1:404354652033:secret:exchange-app-db-secrets-yKlzX0"]
    } ]
}')
```

### Next steps are to provide our pods permissions to view secrets
- pods can get permissions either through underlying worker node IAM roles
- or the *better and secure way is 1. to attach IAM role to serviceaccount and 2. attach serviceaccount to the pod*
    
    `pod <= serviceaccount <= IAM role`

**5. Create OIDC provider for the cluster** - required to link IAM roles to service accounts.
*Optional if it was already created - only 1 per cluster is necessary.*
```
eksctl utils associate-iam-oidc-provider --region="$REGION" --cluster="$CLUSTERNAME" --approve 
```

**6. Use `--namespace` to create SA in custom namespace. Create service account linked to an IAM role,** and attach IAM policy that gives access to the created secret
```
eksctl create iamserviceaccount --name nginx-deployment-sa --region="$REGION" --cluster "$CLUSTERNAME" --attach-policy-arn "$POLICY_ARN" --approve --override-existing-serviceaccounts
```

**7. Create the SecretProviderClass** - this typically has to be created for each secret you're trying link (AWS => K8s secret)
```
kubectl apply -f deploy-with-mounted-files/ExampleSecretProviderClassMountedSecretFile.yaml
```


**8. Deploy consumer pod** - pod that starts using this secret: 
```
kubectl apply -f deploy-with-mounted-files/ExampleDeploymentMountedSecretFile.yaml
```

# To use AWS SecretsManager values as env variables
**9. Delete current SecretProviderClass and create a new one with this file**
```
kubectl apply -f deploy-with-env/ExampleSecretProviderClassMountedSecretEnv.yaml
```

**10. Delete current Deployment and create a new one with this file**
```
kubectl apply -f deploy-with-env/ExampleDeploymentMountedSecretEnv.yaml
```

# Note
optional `secretObjects` field is required if you want to create an actual K8s secret

## The logic
when you mount AWS secret as a file, ASCP doesn't create k8s secret
```
pod <= copies AWS secret values at pod creation
```
when you reference AWS secret value as ENV variables, K8s secret must be created. Therefore you need secretObjects properties.
```
pod <= k8s secret <= AWS secret
```

