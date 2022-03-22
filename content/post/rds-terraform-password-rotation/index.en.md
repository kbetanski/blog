---
title: "Rotate RDS master password with Terraform"
description: "How to rotate previously generated master password for AWS RDS in Terraform."
date: "2022-03-10"
lastmod: "2022-03-18"
image: terraform.png
toc: true
tags:
  - aws
  - rds
  - terraform
  - password rotation
links:
  - title: GitLab CI/CD Schedules Documentation
    description: Documentation for GitLab CI/CD tool for software development using continuous methodologies.
    website: https://docs.gitlab.com/ee/ci/pipelines/schedules.html
---

## Existing state

Let's assume the master password has already been generated in the past. It's
been done entirely in Terraform configuration templates. The code looks somewhat
like this:

```tf
resource "random_password" "rds_password" {
  length           = 32
  special          = true
  // AWS RDS does not allow those characters to be used in the master password
  // 
  override_special = "_%@"
}

resource "aws_rds_cluster" "rds_cluster" {
  cluster_identifier      = "aurora-cluster-demo"
  engine                  = "aurora-mysql"
  engine_version          = "5.7.mysql_aurora.2.03.2"
  availability_zones      = ["us-west-2a", "us-west-2b", "us-west-2c"]
  database_name           = "database"
  master_username         = "user"
  master_password         = random_password.rds_password.result
  backup_retention_period = 5
  preferred_backup_window = "07:00-09:00"
}
```

Let's find the key for `random_password.rds_password` resource in the state:
```bash {linenos=false}
terraform state list | grep "random_password.rds_password"
```

For the article purposes, the key will be exactly:
`random_password.rds_password`. But in the case of more complex infrastructure
configuration, it can be e.g.
`module.rds_cluster_eu_west.random_password.primary_cluster_master_password`.

## Replace the resource

Resource of generated password is already saved in the state of the
infrastructure. Terraform will not attempt to change it, but we can tell it to
do so. It can be achieved using the apply command with the `-replace` flag. 

The final command for our code example is:
```bash {linenos=false}
terraform apply -replace="random_password.rds_password"
```

Terraform will show the plan of required changes in the infrastructure. After
accepting the changes, the AWS RDS cluster will immediately try to change the
master password.

## Automated rotation

Password rotation can be simply automated on the CI/CD of choice. GitLab CI/CD
example:

```yaml
rotate-password:
  script:
    - terraform init
    - terraform apply -input=false -replace="$(terraform state list | grep random_password.rds_password)"
  only:
    - scheduled
```

Define the job above in the `.gitlab-ci.yml` file in the repository and head to
the **CI/CD -> Schedules** in the GitLab project. There can be added a schedule
of how often the CI will be triggered using cron notation. More information is
in the link to the GitLab repository at the end of the article.
