# Corban


## All Things Docker
Deploying a Docker container up to an AWS EC2 cloud instance can
be relatively simple. Creating a pipeline with Terraform seems to be the best way to start with the process of deploying things to cloud. For instance, a `.github/workflows/.TerraformApply.yml` file should be made to define how the Github Actions pipeline should be ran. This can also define things like planning PR approvals, linting to check terraform files before pipeline execution, or even checking for drift in the infrastructure state. Afterwards, a larger and more extensive/specific `main.tf` file can be created for deployment of your container.

### Main.tf
```
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t3.micro"

  root_block_device {
    volume_size = 8
  }

  user_data = <<-EOF
    #!/bin/bash
    set -ex
    sudo yum update -y
    sudo amazon-linux-extras install docker -y
    sudo service docker start
    sudo usermod -a -G docker ec2-user
    sudo curl -L https://github.com/docker/compose/releases/download/1.25.4/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
    sudo chmod +x /usr/local/bin/docker-compose
  EOF

  vpc_security_group_ids = [
    module.ec2_sg.this_security_group_id,
    module.dev_ssh_sg.this_security_group_id
  ]
  iam_instance_profile = aws_iam_instance_profile.ec2_profile_hello_world.name

  tags = {
    project = "hello-world"
  }

  monitoring              = true
  disable_api_termination = false
  ebs_optimized           = true
}
```

This snippit of a `main.tf` shows an example of defining an EC2 instance and running a bootstrap script to deploy the pre-made docker container to the instance.

You can view the full file setup (including database setup and VPC/Networking setup) here: https://klotzandrew.com/blog/deploy-an-ec2-to-run-docker-with-terraform

This process can also be defined in a normal YAML file like this: https://docs.github.com/en/actions/creating-actions/creating-a-docker-container-action


## NuGet Package Pushing
Nuget packages can be pushed/published using Github Actions as well. Through a YAML file, details about the package and how it is pushed can be defined in the `.github/workflows` folder.

### Publish.yml
```
name: NuGet Generation

on:
  push:
    branches:
      - master
  pull_request:
    types: [closed]
    branches:
      - master

jobs:
  build:
    runs-on: ubuntu-18.04
    name: Update NuGet package
    steps:

      - name: Checkout repository
        uses: actions/checkout@v1

      - name: Setup .NET Core @ Latest
        uses: actions/setup-dotnet@v1
        with:
          source-url: https://nuget.pkg.github.com/<organization>/index.json
        env:
          NUGET_AUTH_TOKEN: ${{secrets.GITHUB_TOKEN}}        

      - name: Build solution and generate NuGet package
        run: |  
          cd <project>
          dotnet pack -c Release -o out  

      - name: Push generated package to GitHub registry
        run: dotnet nuget push ./<project>/out/*.nupkg --skip-duplicate --no-symbols true
```
This is an example of a NuGet publish on each push made to the repository.

Link where example code was found: https://stackoverflow.com/questions/57889719/how-to-push-nuget-package-in-github-actions

There is also a Github Action in the Github Marketplace that also supports NuGet building, packaging, and publishing. This can be found here: https://github.com/marketplace/actions/publish-nuget

## Static Website Hosting through Github Pages




## Github Actions Pipelines

As stated in the Docker section, a filepath of `.github/workflows` is needed in order to define Github Actions pipelines. These pipelines can be defined through Terraform files or through YAML files.

### TerraformApply.yml
```
name: Terraform Apply

on: [push]

jobs:
  terraform_apply:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v1


    - name: Install Terraform
      env:
        TERRAFORM_VERSION: "0.12.15"
      run: |
        tf_version=$TERRAFORM_VERSION
        wget https://releases.hashicorp.com/terraform/"$tf_version"/terraform_"$tf_version"_linux_amd64.zip
        unzip terraform_"$tf_version"_linux_amd64.zip
        sudo mv terraform /usr/local/bin/
    - name: Verify Terraform version
      run: terraform --version

    - name: Terraform init
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      run: terraform init -input=false

    - name: Terraform validation
      run: terraform validate

    - name: Terraform apply
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      run: terraform apply -auto-approve -input=false
```

This `TerraformApply.yml` is a simple example of getting Terraform installed in the pipeline, defining that `on: [push]` (on a code push), the pipeline should be ran. Note that this pipeline that has been created must also have a `main.tf` file somewhere in the repository in order to build successfully (can be an empty file).

Here is a simple quickstart guide if you are not using Terraform to define pipelines: https://docs.github.com/en/actions/quickstart


You can find more documentation and helpful examples all in this Github repo: https://github.com/dflook/terraform-github-actions
