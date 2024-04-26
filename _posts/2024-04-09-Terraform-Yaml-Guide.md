---
title: "YAML Magic: Automating Terraform Like a Pro"
date: 2024-04-09 21:00:00 +0200 
categories: [Terraform]
tags: [automation, terraform, yaml, iac, devops]
img_path: /terraform/
image: preview_yaml.jpg
---

Following scenario: You have created an extremely well-designed Terraform that automates your boring and repetitive task, BUT you are the only one who knows how to take advantage of it. 

Lets take this example Terraform where we want to automate the creation of an EntraID group for each department in your company:

```hcl
data "azuread_client_config" "current" {}

resource "azuread_group" "Sales" {
  display_name     = "MySalesGroup"
  owners           = [data.azuread_client_config.current.object_id]
  security_enabled = true
  types            = ["DynamicMembership"]

  dynamic_membership {
    enabled = true
    rule    = "user.department -eq \"Sales\""
  }
}

resource "azuread_group" "HR" {
  display_name     = "MyHRGroup"
  owners           = [data.azuread_client_config.current.object_id]
  security_enabled = true
  types            = ["DynamicMembership"]

  dynamic_membership {
    enabled = true
    rule    = "user.department -eq \"HR\""
  }
}

#etc...
```

## Beginner
The above approach is really not a good one, and how should someone who has never worked with terraform create new groups?
A better approach would be to define the groups as a local variable:

```hcl
locals {
  groups = {
    Sales = {
      name = "MySalesGroup"
    }
    HR = {
      name = "MyHRGroup"
    }
    #etc...
  }
}
```

This way it is enough to define the resource once and then use `for_each` to create multiple instances: 
```hcl
resource "azuread_group" "department_group" {
  for_each = local.groups

  display_name     = each.value.name
  owners           = [data.azuread_client_config.current.object_id]
  security_enabled = true
  types            = ["DynamicMembership"]

  dynamic_membership {
    enabled = true
    rule    = "user.department -eq \"${each.key}\""
  }
}
```

## Yaml Magic
To make it even easier, so that even the trainee in the first year of apprenticeship can create new groups in EntraID, just use a Yaml file. 
Why Yaml you might ask?

> Well, there is litherally nothing simpler than Yaml - Me 2024

If we go with the example above:
```hcl
groups = {
    Sales = {
      name = "MySalesGroup"
    }
    HR = {
      name = "MyHRGroup"
    }
    #etc...
  }
```
A equivalent Yaml file would look like this:
```yaml
Sales:
  name: "MySalesGroup"
HR:
  name: "MyHRGroup"
```
{: file="groups.yml" }

Then you just need to decode the Yaml and use it in your resource the same way as with the local variable:
```hcl
locals {
  groups = yamldecode(file("groups.yml"))
}

resource "azuread_group" "department_group" {
  for_each = local.groups

  display_name     = each.value.name
  owners           = [data.azuread_client_config.current.object_id]
  security_enabled = true
  types            = ["DynamicMembership"]

  dynamic_membership {
    enabled = true
    rule    = "user.department -eq \"${each.key}\""
  }
}
```

## Advanced Magic
You can use Yaml to write just about anything. Want to have a list inside a map inside a map? Sure thing:
```yaml
groups:
  Sales:
    name: "MySalesGroup"
    members:
      - luke
      - leia
  HR:
    name: "MyHRGroup"
    members:
      - han
      - kylo
```
{: file="groups.yml" }

Now you can create groups with members based on the Yaml file:
```hcl
locals {
  yaml_file = yamldecode(file("groups.yml"))
  groups    = local.yaml_file.groups
}

data "azuread_users" "users" {
  for_each = local.groups

  user_principal_names = each.value.members
}

data "azuread_client_config" "current" {}

resource "azuread_group" "group" {
  for_each = local.groups

  display_name     = each.value.name
  owners           = [data.azuread_client_config.current.object_id]
  security_enabled = true
  members          = data.azuread_users.users[each.key].object_ids
}
```` 

> Watch out, as `for_each` supports only maps and sets of strings!
{: .prompt-info }

# Conclusion
And that was just the tip of the iceberg of what it could do. I hope I have shown you how you can use the magic of Yaml to your advantage to simplify your life; or more importantly, the lives of others.
