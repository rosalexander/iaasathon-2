# IaaSathon 2 Walkthrough

In this lab, we were tasked to migrate a custom image from one compartment to another through Terraform and the OCI (Oracle Cloud Infrastructure) CLI.

## Before We Begin

We need to create a file called `terraform.tfvars`. This file will contain sensitive information used to authenticate ourselves to the OCI.

```
//terraform.tfvars

user_ocid = [OCID of the OCI user. We used api.user]
tenancy_ocid = [OCID of the tenancy]
fingerprint = [Fingerprint of OCI public key added to user]
private_key_path= [Path to OCI private key file]
ssh_public_key_path = [Path to public ssh key]
ssh_private_key_path = [Path to private ssh key]
ssh_public_key = [Public key]
region = [Region of tenancy]
customer_compartment_ocid = [OCID of the source compartment]
cloud_compartment_ocid = [OCID of the destination compartment]
instance_ocid = [OCID of the instance to create a custom image of and then migrate]
```

NOTE: All these variables should be in strings using double quotes. The `private_key_path` is the path to the PEM private key generated for users. The `ssh_public_key` should be the actual public SSH key, not the path to it.

These values are passed onto `vars.tf` and is mainly used in `main.tf`.

Also, this walkthrough requires the installation of the OCI CLI. Please have that installed before continuing.

## Terraform Structure

We split our Terraform project into modules, which can be thought of as smaller Terraform functions. A module will be in its own separate folder in the `modules` directory.

```
main.tf
vars.tf
provider.tf
terraform.tfvars
modules
 |
 |---- vcn
 |---- bucket
 |---- create_image
 |---- migrate_image
 |---- compute

```

In `main.tf` we run each module sequentially, starting with the `vcn` module, and ending with the `compute` module. Here is a brief description of what each module does.

`vcn` - Creates a new virtual cloud network, internet gateway, and a subnet with a security list

`bucket` - Creates a new object storage bucket to later migrate an image into

`create_image` - Creates a new custom image with the provided image OCID and in the respective compartment

`migrate_image` - Migrate the created custom image from a bucket onto the source compartment

`compute` - Create a new instance with the recently migrated image.

## Step 0: Configuring the Provider and Main File

Don't forget to configure your `terraform.tfvars` file beforehand!

In this step we will set up our `provider.tf` file to allow us to authenticate into OCI. If we did not have this file, then we would have to run our authentication code for every module we run. This way, we only need to authenticate once. Read the `provider.tf` [file](provider.tf) for a better idea on how to format it. For more information, click [here](https://www.terraform.io/docs/configuration/providers.html) (Note: this link uses AWS in their examples).

Our `main.tf` [file](main.tf) is what we will use to run all our modules. Every time we want to add a module we use this block:

```
module "module_name_1" {
  source = [insert path to folder of module]
  example_variable_1 = "${var.example_variable_1}"
  example_variable_2 = "hard coded variable"
  example_variable_3 = "${module.example_module.example_variable}"
}
```
In this code we set a path to the module and pass in variables the module requires. These variables should be set beforehand `vars.tf` and `terraform.tfvars` (especially if they are sensitive) but you can also hard code them like in `example_variable_2`. There is an example of how to pass in external variables outputted by a module in `example_variable_3`. We will get to that later.

To initialize the Terraform project, call `terraform init` on your command line. To see how the project would change your OCI infrastructure call `terraform plan`. To apply these changes, run `terraform apply`


## Step 1: Creating the VCN

Our code for creating a VCN and subnet was adapted using the template found [here](https://gist.github.com/lucassrg/9b97fb224cb4882d7db6b04a5b048ea8). We open port 80, 3000, 5000, and 1521 because our web application we would migrate needs them. 

In `main.tf` we pass the variables `user_ocid`, `tenancy_ocid`, `compartment_ocid`, `ssh_public_key_path`, `ssh_private_key_path`, `fingerprint`, `region`, and `availability_domain`. 

***IMPORTANT:*** We have hard-coded our `region` variable as "ad-ashburn-1." If your tenancy is in, for example, "us-phoenix-1", then you must change the value of `region`. Furthermore, we have decided as a preference to zero-index our availability domains. For example AD-1 is mapped to 0, AD-2 to 1, and AD-2 to 2. Therefore, since we want to use AD-1, our `availability_domain` variable returns 0.

### A Brief Intro to Output Variables

In `modules/vcn` we also have an `outputs.tf` [file](/modules/vcn/outputs.tf). We are outputting a variable called `subnet_ocid` which we will use later when we compute an instance. This is very helpful, because without the ability to output variables, we would have to run the vcn module, pause to find OCID of the subnet we just created, manually pass it to our compute module, and then run the compute module. By outputting the variable, we can run modules one after another even if one module is dependent on another module. Terraform will understand there is an implicit dependency between those modules (you cannot yet state explicit dependencies between module). We can reference the `subnet_ocid` variable in `main.tf` as `modules.vcn.subnet_ocid`. We will use more output variables later in the tutorial. Read more about output variables [here](https://www.terraform.io/intro/getting-started/outputs.html)

Also learn more about dependencies [here](https://www.terraform.io/intro/getting-started/dependencies.html). They're also important to know!

## Step 2: Creating the Bucket

Bucket creation is quite simple to do once you're familiarized with the Terraform format and navigating the Terraform OCI Provider documentation. We took the code for bucket creation [here](https://github.com/terraform-providers/terraform-provider-oci/blob/master/docs/examples/object_storage/bucket.tf). There is a new type of concept introduced called a data source. According to the [documentation](https://www.terraform.io/docs/providers/external/data_source.html), "data sources allow data to be fetched or computed for use elsewhere in Terraform configuration." They are like easy ways to output variables with useful information. The example code creates an object storage bucket summary data source which can provide data such as the name of the bucket, the compartment it belongs to, and the bucket storage tier. You can read more about the bucket summary data source [here](https://www.terraform.io/docs/providers/oci/d/object_storage_buckets.html).  We did not play around much with them in this lab, but data sources can be incredibly helpful tools.
 
## Step 3: Creating the Custom Image

Our custom image file [here](/modules/create_image/create_image.tf) is super sparse and hopefully is easy to understand. All it requires is the instance OCID, the compartment OCID, and a display name for the newly created image. It will take a little time to create the image depending on how large it is.

## Step 4: Exporting the Custom Image to a Bucket

Unfortunately we did not find an elegant way to export a custom image onto a bucket through Terraform, so we have to step out of it shortly to use the OCI CLI. We played around with using Terraform's [local-exec](https://www.terraform.io/docs/provisioners/local-exec.html) tool to run OCI CLI commands through Terraform, but since the CLI is unable to send a response when the upload to the bucket is complete, we could not reliably depend on it as it could cause dependency issues.

Use the OCI CLI (command line interface) to export an image onto a bucket with this command:

`oci compute image export to-object --image-id [OCID of image] -ns [OCI namespace] -bn [bucket name] --name [name of new object]`

The OCI CLI reference guide can be found [here](https://docs.cloud.oracle.com/iaas/tools/oci-cli/latest/oci_cli_docs/index.html). We recommend learning how to use it; it's actually quite intuitive!

***NOTE***: Our team has decided to just migrate the image just through the OCI ecosystem. Other teams have instead downloaded the image onto their local environment and used Terraform/CLI/API/SDK (there are many ways to do this!) to upload the file from their local machine to a bucket. A Terraform way can be found [here](https://www.terraform.io/docs/providers/oci/r/object_storage_object.html#) and by providing the `source` variable.

## Step 5: Migrating the Custom Image from Bucket to Compute

We used the example called "Create image from exported image via direct access to object store" [here](https://www.terraform.io/docs/providers/oci/r/core_image.html). We also outputted the OCID of the image for creating the compute instance later. Hope this is clear!

## Step 6: Creating a Compute Instance

Finally, we create the compute mostly using code from Abhiram Ampabathina [here](https://github.com/mrabhiram/terraform-oci-sample/tree/master/modules/compute-instance) (we barely wrote any original code as you can probably tell, but we never really tread any new ground that required new code. As long as you have a good understanding of Terraform, we believe it's okay. And even if you don't, looking at example code is a good way to learn ☺️).

## Conclusion
It has been made clear through this lab how powerful and useful Terraformis is for DevOps and cloud developers. If we had more time, we would like to find a solution to download files on OCI and upload them all through Terraform (maybe by utilizing the OCI Python SDK?). Anyway, this IaaSathon was a great learning experience and deep dive into Terraform. We hope this walkthrough was useful!
