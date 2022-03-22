---
title: "Rotacja hasła administratora AWS RDS używając Terraforma"
description: "Jak zmienić poprzednio wygenerowane hasło administratora w AWS RDS przy użyciu Terraforma."
date: "2022-03-18"
lastmod: "2022-03-18"
image: terraform.png
toc: true
tags:
  - aws
  - rds
  - terraform
  - password rotation
links:
  - title: GitLab CI/CD Schedules Dokumentacja
    description: Dokumentacja dla narzędzia GitLab CI/CD.
    website: https://docs.gitlab.com/ee/ci/pipelines/schedules.html
---

## Obecny stan

Przyjmijmy, że hasło administratora zostało wcześniej wygenerowane przy użyciu
skryptów Terraform. Kod takiego skryptu wygląda następująco:

```tf
resource "random_password" "rds_password" {
  length           = 32
  special          = true
  // AWS RDS nie pozwala na użycie tych znaków specjalnych dla głównego hasła
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

Do znalezienia jest w tym przypadku klucz, pod którym znajduje się zapisany
zasób 'random_password.rds_password' w zapisanym stanie:

```bash {linenos=false}
terraform state list | grep "random_password.rds_password"
```

Przyjmijmy, że klucz ma nazwę: 'random_password.rds_password'. Jednakże, w
przypadku bardziej skomplikowanych skryptów klucz może posiadać również inne
nazwy, na przykład:
'module.rds_cluster_eu_west.random_password.primary_cluster_master_password'.

## Zastąpienie starego zasobu nowym

Utworzony wcześniej zasób wygenerowanego hasła jest zapisany w pliku stanu
infrastruktury. Terraform nie podejmie się wygenerowania tego hasła ponownie ze
względu na to, że zasób hasła został już utworzony. Należy wymusić na nim
ponowne wygenerowanie tego zasobu przy wykorzystaniu flagi '-replace'.

Podsumowując, komenda generująca nowe hasło główne wygląda tak:

```bash {linenos=false}
terraform apply -replace="random_password.rds_password"
```

Terraform przedstawi plan zmian do przeprowadzenia w infrastrukturze. W naszym
przypadku będzie do zastąpienie zasobu hasła administratora nowym oraz zmiana
wartości przekazywanej do pola 'master_password' w
'aws_rds_cluster.rds_cluster'. Po zaakceptowaniu zmian do wdrożenia klaster AWS
RDS natychmiastowo przejdzie do próby zmiany hasła administratora całym
klastrze.

## Automatyzacja rotacji hasła

Rotacja hasła może być w prosty sposób zautomatyzowana w dowolnym narzędziu dla
CI/CD. Przykład konfiguracji dla GitLab CI/CD:

```yaml
rotate-password:
  script:
    - terraform init
    - terraform apply -input=false -replace="$(terraform state list | grep random_password.rds_password)"
  only:
    - scheduled
```

Po zdefiniowaniu takiej konfiguracji w pliku '.gitlab-ci.yml' w repozytorium
trzeba jeszcze skonfigurować cykliczne uruchamianie CI/CD. Odpowiednie
ustawienia można znaleźć w menu projektu pod **CI/CD -> Schedules**. W tym
miejscu można zdefiniować jak często CI/CD ma być wywoływany do uruchomienia
przy użyciu zapisu narzędzia 'cron'. Więcej informacji można znaleźć w
dokumentacji podlinkowanej na końcu artykułu.
