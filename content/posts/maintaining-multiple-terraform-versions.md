---
title: "Maintaining Multiple Terraform Versions"
date: 2021-12-21T21:16:35+05:30
draft: false
---

While for most cases the latest version is enough, depending on the requirements, and providers, sometimes we have to keep a specific older version of Terraform binary handy to manage our infrastructure. Here are some ways users can maintain multiple versions easily on their desktops locally or on servers using third-party tools or manually.

## Third-party tools

[Terraform Switcher (tfswitch)](https://github.com/warrensbox/terraform-switcher) and [tfenv](https://github.com/tfutils/tfenv) both similar tools that lets users pick and use a specific Terraform version they or their code requires. Both can automatically pick the version needed, or users can specify the version they need. Other than that, from user perspective, they do not have to do any manual set up to get the Terraform version up and running. The tools themselves will download, extract, and place them in the correct paths so when the users run Terraform commands as usual, the correct version is executed. I personally prefer `tfswitch` though.

### Maintaining specific versions manually

Sometimes users might have to use a specific version hard coded. In such cases, it would be easier to set it up manually without using a third-party tool. Especially if they set up Terraform on a VM which itself is going to be provisioned through infra-as-code, it would be wiser to have a way to maintain a specific binary themselves.

Here is an example script which you can use to set up a specific Terraform quickly. You can modify the paths, and version as needed, and this can be repeated through a loop if you need more than two versions. In this example, I’m only setting up version 0.14.0 and 1.0.10.

Links to download a specific version of Terraform binary can be found here https://releases.hashicorp.com/terraform/ Or you can use a local repo to hold the binaries yourselves too, to make sure the availability.

```bash
#! /bin/bash
versions=('0.14.0' '1.0.10')

for v in ${versions[@]}; do
    mkdir -p /usr/local/terraform/${v}
    cd /usr/local/terraform/${v}
    wget https://releases.hashicorp.com/terraform/${v}/terraform_${v}_linux_amd64.zip
    unzip terraform_${v}_linux_amd64.zip
    rm -f terraform_${v}_linux_amd64.zip
    binVersion=$(echo "$v" | sed "s/\.//g")
    ln -s /usr/local/terraform/${v}/terraform /usr/bin/tf${binVersion}
done
```

Running `tf0140 -v` and `tf1010 -v` should give you the corresponding versions correctly.
