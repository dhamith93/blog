---
title: "Generating Ansible Inventory Files From Terraform"
date: 2021-12-10T21:12:27+05:30
draft: false
---

First a template of the inventory file should be created depending on the infrastructure design with all the dynamic values set as tokens.

For example, here is an inventory file template which handles multiple EC2 instances created in AWS. Server IP and private key file path is added as tokens and it will be filled once the terraform finish creating the infrastructure in AWS or any other providers. For this example AWS is used.

```ini
[db_server]
${db_ip} ansible_user=${user} ansible_ssh_private_key_file=${key_path}

[dev_server]
${dev_ip} ansible_user=${user} ansible_ssh_private_key_file=${key_path}

[prod_server]
${prod_ip} ansible_user=${user} ansible_ssh_private_key_file=${key_path}
```

Then output.tf file should be created/edited in terraform to create the final inventory file by filling the tokens in the template file.

```tf
resource "local_file" "unique_output_name" {
  content = templatefile("template_file.tmpl", {
    token = resource.value
  })
  filename = "output_file"
}
```

For example, here is the output file which will use above template file and generate the inventory file with the public IP of the instance, and the key file path to SSH in to the instance.

```tf
resource "local_file" "ansible_inventory" {
  content = templatefile("inventory.tmpl", {
    db_ip    = module.db_server.public_ip,
    dev_ip   = module.dev_server.public_ip,
    prod_ip  = module.prod_server.public_ip,
    user     = var.user_name,
    key_path = var.key_path
  })
  filename = "inventory"
}
```

The above resource block with type `local_file` will load the inventory.tmpl file and replace the tokens in the template using outputs from module output and from an input variable. As per the above example, variables or output values from modules, or individual resources can be used to fill the token values in the template. Also, a single token can be used multiple times, and replaced with a single value, just like the `key_path` token.

Once the instances and/or other infrastructure is successfully created using terraform, the inventory file will be generated. For example, the final result for above template would be something like this:

```ini
[db_server]
3.2.1.103 ansible_user=centos ansible_ssh_private_key_file=/Users/name/.keys/user-us-east.cer

[dev_server]
3.2.1.104 ansible_user=centos ansible_ssh_private_key_file=/Users/name/.keys/user-us-east.cer

[prod_server]
3.2.1.105 ansible_user=centos ansible_ssh_private_key_file=/Users/name/.keys/user-us-east.cer
```

Once it is generated, any ansible operation can be done using that inventory file. For example,

```bash
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook main.yml -i ../terraform/inventory
```

Above command will let an Ansible Playbook to be run without individually adding the IPs to known host list.

For a full example of Ansible, and Terraform in action, this github repo can be referred: https://github.com/dhamith93/terraform_ansible_example

