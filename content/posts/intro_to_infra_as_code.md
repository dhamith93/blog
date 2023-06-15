+++
title = "Introduction to Infrastructure as Code (IaC)"
date = "2023-06-15T20:23:01+05:30"
author = "Dhamith Hewamullage"
authorTwitter = "" #do not include @
cover = "/cover-iac.jpg"
tags = ["DevOps", "IaC", "Infrastructure as Code", "Terraform", "Pulumi"]
keywords = ["DevOps", "IaC", "Infrastructure as Code", "Terraform", "Pulumi"]
description = "Introduction to Infrastructure as Code (IaC)"
showFullContent = false
readingTime = true
hideComments = false
+++

## Background

Imagine you are tasked with deploying an application. In the initial requirements, you see that it is still in the early stages with a small amount of projected user base. And the recommended infrastructure specification is not that complex. Your team currently does not use any Infrastructure as Code tools other than some in-house scripts. The requirements are simple enough that you quickly provision the resources and deploy the application with its dependencies manually. Everything is working perfectly. 

However, with time, the application grows along with its infrastructure requirements. Even though the constant upgrades, bug fixes, and application specifications are well documented, changes to the infrastructure are not. The infrastructure changes are done by everyone on the team and not everyone knows why certain changes were done, and maybe undone along the way. And suddenly when you are reviewing the provisioned resources, you wonder why a server provisioned with 16GB of memory on documentation now has 32GB. 

Although some parts of that scenario might not be very relevant and might be an extreme case in the days of containerization and Kubernetes. It was very relevant when Infrastructure as Code is not in the mainstream. It is still relevant today if you are in a position where you have to provision and or manage Infrastructure resources either on-permisses or in the cloud. Doing things manually will take more time, and can be more error-prone, not repeatable, and inconsistent. Even if your team is doing well-documented deployments, a single ad-hoc change can make it irrelevant. 

## What are the benefits of IaC?

With Infrastructure as Code, provisioning new resources or managing existing resources becomes more efficient and organized. Instead of manually doing the initial provisioning or maintenance, resources are defined in code and changed in code. Changes to that code are tracked using a version-controlling system like git. Anyone who read the code can see a detailed view of what is deployed where. It is easy to audit resources and figure out what kind of resources are under your responsibility. In the repo history, you can see the reason behind the change. Just like the application which is running on that infrastructure. 

Let's review some major benefits of Infrastructure as Code brings.

#### Consistency

Coding your infrastructure makes you more consistent and makes the result reproducible. Similar to running the same code again and again would run the same application, when you run the same infra code in a new environment, you will get the same infrastructure resources created again. For example, you can use the same code base to create dev, QA, prod, and DR environments without worrying about any misconfigurations on the way.

#### Version Control

Just like the applications you are deploying, you can maintain your infrastructure as code, in a git repo along with your application code. Just like a software process, you can use branches to deploy the infrastructure in dev, QA, or prod environments. Create and review pull requests before merging them to the master branch. Instantly roll back on any changes if any issues come up with the latest change you pushed. 

Version control systems also can be used to track and audit currently deployed resources. With good commit messages, you can track every single change you and your team doing to the real-world infrastructure resources. 

This not only helps people individually to reduce their workload. This will help the whole team of engineers and other stakeholders who deal with infrastructure to collaborate on a project very easily and efficiently. With everything being documented with code, comments, commits, and ReadMe's, everyone will be on the same page regarding what exactly are the resources that are online at any given time without logging in and checking multiple environments. 

#### Automation

With your infrastructure moved to a code base that is stored in a version control system, you can easily get the extended benefits of automation. It can either be by any in-house script or a tool that is running the infrastructure code after a change, or it can be done by a CICD pipeline. Where once you push the changes and create a merge request, which will trigger a CICD job to run the infrastructure code after the merge request is approved. 

Another major benefit of automating your infrastructure provisioning is the reduction of human errors. You will still be relying on the quality of the code, but it will reduce mistakes that can happen during the provisioning. With the correct tools, mistakes can be recognized even before the code is executed and resources are provisioned. Giving you plenty of chances to fix the issue with the code before the actual resources are created. 

Overall, with automation, the provisioning will be robust and less error-prone. With automated checks, verifications, and pipeline jobs, the time it will take for the provisioning will be reduced as well as the human intervention during provisioning can be reduced to 0 with a correct automation pipeline.

#### Scalability

Scaling up and down manually takes time, needs verifications, and if you have multiple resources bound together, makes things complex. However, with Infrastructure as Code, you have total control and flexibility over the resources you have deployed. As per the dynamic workload demands you can easily adjust the resources and deploy the changes. This might be new instances to handle the number of requests your application is getting. Or just increasing the storage in the existing instance to deal with low disk space alerts. Manually doing that also keeps your team in the dark on the latest specification of the provisioned instances. However with IaC in a git repo with changes with meaningful commits, will show every stakeholder the current state as well as the reason behind the change.

## Available IaC tools

There are several Infrastructures as Code tools available. All of them do the same thing more or less but in different ways. Following is a list of a few popular IaC tools. I did not include Configuration management tools such as Ansible because, while they do support infrastructure provisioning and management, they are most suited for managing the resources once they are provisioned. 

#### Terraform

[Terraform](https://www.terraform.io/) is an Open Source IaC tool by HashiCorp. It supports all major cloud providers such as AWS, Azure, GCP, and many more service providers. It has good community support and community-maintained providers as well. It also supports multi-cloud and hybrid environments, as well as automation and CICD. Terraform provides flexibility and customizability and it follows a simple workflow when provisioning resources. 

```
Initialize -> Plan -> Apply/Destroy
```

Terraform uses HashiCorp Configuration Language (HCL) to define the infrastructure resources. You can define providers, resources, variables, and other configurations using HCL. Once the code base is initialized, you can run the plan command to see what resources are getting built, and what resources are getting modified or removed. It also validates your code so if there are any errors in your code, you would know it before actually applying the changes. Finally, you can execute the apply/destroy command to push the changes. It also supports workspaces, which allow you to separate environments when provisioning. 

Terraform import feature allows you to import manually created resources into Terraform code base. If you are moving forward with Terraform in your team, you can import already existing resources into your new codebase.

Terraform keeps track of states of resources it is provisioning and their dependencies. It helps to monitor the status before applying any changes to those resources. It also provides state locking so when multiple people are using the same repository to provision/modify resources, to make sure only one person can do the changes at a time. This prevents any conflicts from happening. Usually, the state file is stored locally, but Terraform supports remote backends. This means you can store the state file at a shared location such as an AWS S3 bucket. Making it easier to collaborate with a team.

Another important feature Terraform offers is the Terraform modules. Modules are a way of grouping multiple resources as a single entity. It makes a complex collection of resources easier to manage, share and reuse within the same project or in another project. This prevents any duplication of complex code across multiple code bases and gives a single place to do any changes when necessary. To use a module, all you have to do is import the module and create new resources, filing the necessary inputs as variables. 

#### Pulumi

[Pulumi](https://www.pulumi.com/) takes a different approach to Infrastructure as Code than most other IaC tools. Pulumi supports multiple programming languages (TypeScript, Python, Go, C#, Java) as well as YAML. With the programming language support comes the ability to use complex logic when provisioning resources than just using YAML or JSON. That also means that you can use a language that you are familiar with to write your infrastructure code. You can use a standard IDE to write and debug your code as well.

Just like Terraform Pulumi is also Open Source. Supports all major cloud service providers. Support multi-cloud and hybrid. Has good support for CICD and version control. By default, Pulumi also uses a local state file with support to a remote backend like an AWS S3 bucket to store the state file. Pulumi has a state locking to make sure that the resources are being modified or provisioned by a single deployment at a time. 

#### AWS CloudFormation, Azure Resource Manager, GCP Cloud Deployment Manager

All three of these IaC tools are first-party solutions offered by their respective cloud service providers to automate resource provisioning. While they provide great compatibility with their services in their environments, if you want to migrate to another service provider one day, relying on these tools makes it harder. You would have to migrate your IaC code base to another tool as well. This also means somewhat reduced community support, and less flexibility and customizability. But great compatibility with their resources means that they can provide much better integration into their platforms. Also, if any changes are being done to their platforms, these tools will have a quicker adaptability for those changes.  

## Best practices and pitfalls of IaC

Infrastructure as Code is another approach to make your life as an engineer easier. However, it is easy to misuse IaC tools when you use them day-to-day. Whether you are working alone, or working with a team to provision and manage resources, it is important to know that you have to be consistent with your Infrastructure code. It should maintain good readability. The code should be well documented. Code should be reviewed before execution at least in the production environment. After all, simple misconfiguration here would either break down the deployed infrastructure immediately, if you are lucky or it would take a single specific case, down the line where you have to go through code that you wrote months earlier. 

Make sure your code is modular. It will make your complex combination of resources easy to manage. And once you create a single unit module, you can reuse it across your projects. It will also help you to have a coding standard within your team. Avoid using hardcoded values as much as possible. Make use of parameterized values. That will make sure that your code is usable across multiple environments. 

Test your code. Not only for syntax errors but also for misconfigurations. It is easy to mistake one instance type for another. Especially if you are copy-pasting code from another place. You always have to make sure that the code goes through the testing phase before being pushed into the master branch. Especially if you have CICD pipelines set up to automatically deploy the changes. If possible, set up a dev environment to test infrastructure code. Validate end result on the dev environment before pushing it to the production.

Set up the CICD pipeline to test, validate, and deploy your changes. This avoids any challenges team members will face when they try to deploy changes locally. You don't have to give full admin access to each individual. Instead, you can follow the principle of least privilege. Testing and validation can be done automatically before executing the code. Which will make sure that the code that is being executed on the master branch is tested and validated. Automated testing along with code review processes, will increase the efficiency of maintaining and provisioning infrastructure greatly. 

You have to avoid doing any changes to the resources managed by IaC, manually. There should be processes in place to handle quick requests for changes if needed through the code. When you doing any changes manually while the code does not reflect that change, you are creating a configuration drift. The resource you have in the code will be different from the one you have deployed in the cloud. Which makes auditing and reviewing hard. And on sudden issues with the deployed application or the resources, not all team members will be on the same page regarding the infrastructure the application is running on.

#### Pitfalls

Infrastructure as Code comes with its pitfalls. IaC tools come with a learning curve that might take time for everyone in your team to get used to. Especially if you are migrating existing resources into the IaC code base. Just like managing an application on a version-controlling system, managing infrastructure code means that you have to deal with multiple team members working on the same code base which could lead to merge conflicts. Handling complex infrastructure will require good planning and documentation within the organization. Code will be more complex and hard to maintain if they are not very well defined or documented from the beginning. You will have to have a good way of testing and validating the code before executing them. Not only the code, but you also have to validate that the infrastructure created by the code is up to spec. And in the case of tools like AWS CodeFormation, you have the risk of being vendor locked in. 

All of that means that you have to invest your time when picking the tool you are going to use. You have to invest time to make sure all your team members understand IaC concepts and the tool you have picked. Make sure they know the best practices as per the cloud service provider and the tool you have picked. You have to document everything. Make sure no one manually changes the resources once they are provisioned. Review the code for errors, misconfigurations, and security vulnerabilities.
