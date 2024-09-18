
Following script provides high level information about the types of resources and their count in a given AWS account:

```
#!/bin/bash

# Function to compute and print the count of AWS resources for a given region
compute_resource_counts() {
  local region=$1

  echo "Computing resource counts for region: $region"

  # IAM Roles
  iam_roles_count=$(aws iam list-roles --query 'Roles[*].RoleName' --output json --region $region | jq length)

  # IAM Policies
  iam_policies_count=$(aws iam list-policies --query 'Policies[*].PolicyName' --output json --region $region | jq length)

  # EC2 Instances
  ec2_instances_count=$(aws ec2 describe-instances --query 'Reservations[*].Instances[*].InstanceId' --output json --region $region | jq 'flatten | length')

  # EKS Clusters
  eks_clusters_count=$(aws eks list-clusters --query 'clusters' --output json --region $region | jq length)

  # ECS Clusters
  ecs_clusters_count=$(aws ecs list-clusters --query 'clusterArns' --output json --region $region | jq length)

  # ECS Task Definitions
  ecs_task_definitions_count=$(aws ecs list-task-definitions --query 'taskDefinitionArns' --output json --region $region | jq length)

  # Lambda Functions
  lambda_functions_count=$(aws lambda list-functions --query 'Functions[*].FunctionName' --output json --region $region | jq length)

  # RDS Instances
  rds_instances_count=$(aws rds describe-db-instances --query 'DBInstances[*].DBInstanceIdentifier' --output json --region $region | jq length)

  # Security Groups
  security_groups_count=$(aws ec2 describe-security-groups --query 'SecurityGroups[*].GroupId' --output json --region $region | jq length)

  # ECR Repositories
  ecr_repositories_count=$(aws ecr describe-repositories --query 'repositories[*].repositoryName' --output json --region $region | jq length)

  # Total Number of ECR Images across Repositories
  ecr_image_count=0
  for repository in $(aws ecr describe-repositories --query 'repositories[*].repositoryName' --output text --region $region); do
    images=$(aws ecr list-images --repository-name $repository --query 'imageIds[*].imageDigest' --output json --region $region | jq length)
    ecr_image_count=$((ecr_image_count + images))
  done

  # Print the results for the region
  echo "Results for region $region:"
  echo "IAM Roles: $iam_roles_count"
  echo "IAM Policies: $iam_policies_count"
  echo "EC2 Instances: $ec2_instances_count"
  echo "EKS Clusters: $eks_clusters_count"
  echo "ECS Clusters: $ecs_clusters_count"
  echo "ECS Task Definitions: $ecs_task_definitions_count"
  echo "Lambda Functions: $lambda_functions_count"
  echo "RDS Instances: $rds_instances_count"
  echo "Security Groups: $security_groups_count"
  echo "ECR Repositories: $ecr_repositories_count"
  echo "Total ECR Images: $ecr_image_count"
  echo "---------------------------------------------"
}

# Main script logic

# Check if regions are passed as input
if [ -z "$1" ]; then
  echo "Usage: $0 <comma-separated-region-names>"
  exit 1
fi

# Split the input regions into an array
IFS=',' read -ra regions <<< "$1"

# Iterate over each region and compute resource counts
for region in "${regions[@]}"; do
  compute_resource_counts "$region"
done
```

Steps to execute the script: 
1. Save the script.(Ex: aws_resource_count.sh)
2. Make the script executable
```
chmod +x aws_resource_count.sh
```
3. Run the script
```
./aws_resource_count.sh us-east-1,us-west-1,eu-central-1
```


