# grafana-loki-s3-configuration
Configuration of S3 storage to grafana loki
grafana-loki-S3-configuration setup
-------------------------------------------------------------------------------
 # create a role for s3
add the policy
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject"
            ],
            "Resource": "arn:aws:s3:::dev-eks-loki-logs/*"
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::dev-eks-loki-logs"
        },
        {
            "Sid": "VisualEditor2",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:ListBucket",
                "s3:DeleteObject"
            ],
            "Resource": "arn:aws:s3:::dev-eks-loki-logs"
        }
    ]
}
# in the policy add the trust relationship
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::6648xxxxx:oidc-provider/oidc.eks.ap-south-1.amazonaws.com/id/B5F325760F6FE1F194517C9DA9399086"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.ap-south-1.amazonaws.com/id/B5F325760F6FE1F194517C9DA9399086:sub": "system:serviceaccount:default:s3-echoer"
                }
            }
        },
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::6648xxxxx:oidc-provider/oidc.eks.ap-south-1.amazonaws.com/id/B5F325760F6FE1F194517C9DA"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "oidc.eks.ap-south-1.amazonaws.com/id/B5F325760F6FE1F194:sub": "system:serviceaccount:loki:loki-sa"
                }
            }
        }
    ]
}


--------------------------------------------------------------------------------------------------------------------------------------
# find the OIDC Number add in the above trust relatioship JSON
to find the OIDC command is

oidc_id=$(aws eks describe-cluster --name Inspz-eks \
--query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)

----------------------------------------------------------------------------------------------------------------------------------------
execute the commands

eksctl utils associate-iam-oidc-provider --region ap-south-1 --cluster Inspz-eks --approv

--------------------------------------------------------------------------------------------------------------------------------------
# create the values.yaml file for the S3 Configuration
with these contents


loki:
  auth_enabled: false
  commonConfig:
    path_prefix: /var/loki
    replication_factor: 1
  compactor:
    apply_retention_interval: 1h
    compaction_interval: 5m
    retention_delete_worker_count: 500
    retention_enabled: true
    shared_store: s3
    working_directory: /data/compactor
  config:
      schema_config:
        configs:
        - from: 2020-05-15
          store: boltdb-shipper
          object_store: s3
          schema: v11
          index:
            period: 24h
            prefix: loki_index_

      storage_config:
        aws:
          region: ap-south-1
          bucketnames: dev-eks-loki-logs
          s3forcepathstyle: false
          #s3forcepathstyle: true  <-- This is the main culprit; comment it out ? -? https://github.com/grafana/loki/issues/7024
        boltdb_shipper:
          shared_store: s3
          cache_ttl: 24h
  serviceAccount:
    create: true
    name: loki-sa
    annotations:
       eks.amazonaws.com/role-arn: "arn:aws:iam::6648xxxxx:role/loki-s3-role"
  write:
     replicas: 2
  read:
    replicas: 1


--------------------------------------------------------------------------------------------------------------
# execute this commands

helm upgrade -f values.yaml loki grafana/loki-stack -n namespace

arn:aws:iam::6648xxxxx:role/eks-cluster-role 




