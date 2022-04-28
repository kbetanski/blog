---
title: "Terraform - reusable templates"
description: "Using modules to create reusable templates to omit repetition"
date: "2022-04-23"
lastmod: "2022-04-23"
toc: true
draft: true
slug: terraform-reusable-templates
tags:
  - terraform
  - templates
  - modules
  - IaaC
links:

---

Most of the programmers know the DRY principle, which stands for Don't Repeat
Yourself. It's a principle that states that you should avoid duplications in
your code. It's a logical decision to use it for the IaaC as well. While working
with complex infrastructure configuration, it's often the case that you need to
repeat the same code over and over again.

In Terraform it's possible to create reusable modules that can be used in
different places. Also, it's possible to create templates for declaring reusable
scripts depending on the arguments passed to the template function. But what
about reusing the same template file in different modules at once? I would like
to share what I taught myself about reusing templates in Terraform.

## Declaring module

Terraform modules are really easy to declare and use. To create one you need to
create a separate catalog with terraform files, with extension `.tf`. Let's
start with directory structure like below:

```bash {linenos=false}
main.tf
modules/template/script.sh.tmpl
modules/template/variables.tf
modules/template/outputs.tf
```


```terraform 

```

### 

### Passing arguments

### Using module

## Summary
