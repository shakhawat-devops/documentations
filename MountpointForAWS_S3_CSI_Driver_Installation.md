# Deploy the Mountpoint for Amazon S3 driver on EKS
With the Mountpoint for Amazon S3 Container Storage Interface (CSI) driver, your Kubernetes applications can access Amazon S3 objects through a file system interface, achieving high aggregate throughput without changing any application code.

This procedure will show you how to deploy the Mountpoint for Amazon S3 CSI Amazon EKS driver.

1. Create an IAM Policy

    The Mountpoint for Amazon S3 CSI driver requires Amazon S3 permissions to interact with your file system. This section shows how to create an IAM policy that grants the necessary permissions.

    The following example policy follows the IAM permission recommendations for Mountpoint. Alternatively, you can use the AWS managed policy AmazonS3FullAccess, but this managed policy grants more permissions than are needed for Mountpoint. Name the policy as ``AmazonS3CSIDriverPolicy``
    ```
    {
   "Version": "2012-10-17",
   "Statement": [
        {
            "Sid": "MountpointFullBucketAccess",
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::<your-bucket-name>"
            ]
        },
        {
            "Sid": "MountpointFullObjectAccess",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:AbortMultipartUpload",
                "s3:DeleteObject",
                "s3:ListObject"
            ],
            "Resource": [
                "arn:aws:s3:::<your-bucket-name>/*"
            ]
        }
    ]
    }
    ```
    Replace ``<your-bucket-name>`` with the name of your S3 bucket.

2. Create an IAM role

    The Mountpoint for Amazon S3 CSI driver requires Amazon S3 permissions to interact with your file system. This section shows how to create an IAM role to delegate these permissions. To create this role, We will use AWS-CLI:
    First Create a policy and and an IAM role:
    
    Get the OIDC provider URL: Replace ``<my-cluster>`` with your new one.
    ```
    aws eks describe-cluster --name <my-cluster> --query "cluster.identity.oidc.issuer" --output text
    ```
    Copy the following contents to a file named ``aws-s3-csi-driver-trust-policy.json``
    ```
    {
    "Version": "2012-10-17",
    "Statement": [
        {
        "Effect": "Allow",
        "Principal": {
            "Federated": "arn:aws:iam::111122223333:oidc-provider/oidc.eks.region-code.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE"
        },
        "Action": "sts:AssumeRoleWithWebIdentity",
        "Condition": {
            "StringEquals": {
            "oidc.eks.region-code.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:sub": "system:serviceaccount:kube-system:s3-csi-driver-sa",
            "oidc.eks.region-code.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE:aud": "sts.amazonaws.com"
            }
        }
        }
    ]
    }
    ```
    Replace ``oidc.eks.region-code.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE`` this part with the value returned in the previous step omitting the https.

    Create the role named ``AmazonEKS_S3_CSI_DriverRole``
    ```
    aws iam create-role \
    --role-name AmazonEKS_S3_CSI_DriverRole \
    --assume-role-policy-document file://"aws-s3-csi-driver-trust-policy.json"

    ```

    Attach the previously created IAM policy to the role with the following command.
    ```
    aws iam attach-role-policy \
    --policy-arn <Policy_ARN> \
    --role-name AmazonEKS_S3_CSI_DriverRole
    ```
    Replace ``<Policy_ARN>`` by the ARN of the policy created at step 1.

3. Create the Kubernetes service account in your cluster say like ``s3-csi-driver-sa``. You need to annotate the Service Account with the ARN of the IAM role created in the previous step which will be done while installing the controller.

4. Install the Mountpoint for AWS S3 CSI Driver
We will install the Mountpoint for Amazon S3 CSI driver through the AWS EKS add-on using EKSCTL. Run the below command. Replace my-cluster with the name of your cluster, 111122223333 with your account ID, and AmazonEKS_S3_CSI_DriverRole with the name of the IAM role created earlier. This command will annotate the service account with the ARN of the role created. 

    ```
    eksctl create addon --name aws-mountpoint-s3-csi-driver --cluster <cluster-name> \
    --service-account-role-arn arn:aws:iam::111122223333:role/AmazonEKS_S3_CSI_DriverRole --force
    ```
5. Now Create a Persistent Volume and Persistent Volume Claim. You can use the same yaml snippet below. 

    ```
    apiVersion: v1
    kind: PersistentVolume
    metadata:
    name: s3-pv
    spec:
    capacity:
        storage: 100Gi
    accessModes:
        - ReadWriteMany # Supported options: ReadWriteMany / ReadOnlyMany
    storageClassName: "" # Required for static provisioning    
    mountOptions:
        - allow-delete
        - prefix production/
        - region <your-region>
    csi:
        driver: s3.csi.aws.com # Required
        volumeHandle: s3-csi-driver-volume # Must be unique
        volumeAttributes:
        bucketName: <your-s3-bucket-name>
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
    name: s3-pvc
    spec:
    accessModes:
        - ReadWriteMany # Supported options: ReadWriteMany / ReadOnlyMany
    storageClassName: "" # Required for static provisioning
    resources:
        requests:
        storage: 120Gi # Ignored, required
    volumeName: s3-pv # Name of your PV
    ```
    The whole point of this document is to mount S3 as a volume into the Deployments or Pods. So to provision the volume from S3 you need to have an S3 bucket. Either you can create the bucket manually or use AWS S3 controller for Kubernetes to provision the bucket using native k8s yaml configuration.

    Replace ``<your-region``>, ``<your-s3-bucket-name>`` with your custom values. 

6. Now you can mount this PVC into your Deployment and Pods as volumes. 

