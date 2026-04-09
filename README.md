# Day-62----Providers-Resources-and-Dependencies

# Task

Yesterday you created standalone resources. But real infrastructure is connected -- a server lives inside a subnet, a subnet lives inside a VPC, a security group controls what traffic gets in. Today you build a complete networking stack on AWS and learn how Terraform figures out what to create first.

Understanding dependencies is what separates a Terraform beginner from someone who can build production infrastructure.

# Expected Output

- A VPC with subnet, internet gateway, route table, security group, and an EC2 instance -- all created via Terraform
- A dependency graph visualized with terraform graph
- A markdown file: day-62-providers-resources.md

# Challenge Tasks

# Task 1: Explore the AWS Provider

1. Create a new project directory: terraform-aws-infra
2. Write a providers.tf file:
   - Define the terraform block with required_providers pinning the AWS provider to version ~> 5.0
   - Define the provider "aws" block with your region
3. Run terraform init and check the output -- what version was installed?
4. Read the provider lock file .terraform.lock.hcl -- what does it do?

<img width="967" height="401" alt="Image" src="https://github.com/user-attachments/assets/79908af1-b0df-4bc9-9a22-9a30fa017eef" />

<img width="888" height="108" alt="Image" src="https://github.com/user-attachments/assets/5870edd4-f509-4a01-903f-2d53bbc94e39" />

<img width="962" height="144" alt="Image" src="https://github.com/user-attachments/assets/2f3bbe73-51ba-44c1-92f4-7a9840e57fe3" />

# Task 2: Build a VPC from Scratch

Create a main.tf and define these resources one by one:

1. aws_vpc -- CIDR block 10.0.0.0/16, tag it "TerraWeek-VPC"
2. aws_subnet -- CIDR block 10.0.1.0/24, reference the VPC ID from step 1, enable public IP on launch, tag it "TerraWeek-Public-Subnet"
3. aws_internet_gateway -- attach it to the VPC
4. aws_route_table -- create it in the VPC, add a route for 0.0.0.0/0 pointing to the internet gateway
5. aws_route_table_association -- associate the route table with the subnet
   
Run terraform plan -- you should see 5 resources to create.

<img width="877" height="324" alt="Image" src="https://github.com/user-attachments/assets/e685ed0f-6b69-448b-8fca-62e183da4edb" />

<img width="896" height="330" alt="Image" src="https://github.com/user-attachments/assets/cbb207ee-1a3d-43bf-b41b-41f38e5fb7b5" />

<img width="792" height="289" alt="Image" src="https://github.com/user-attachments/assets/9334e9f4-6d8b-434a-adfe-f54cc080f8cf" />

<img width="965" height="411" alt="Image" src="https://github.com/user-attachments/assets/410ffb95-6e18-4a44-8258-e24fa2aef360" />

<img width="959" height="358" alt="Image" src="https://github.com/user-attachments/assets/a9b022c2-3890-4586-ae48-48365208f875" />

# Task 3: Understand Implicit Dependencies

Look at your main.tf carefully:

1. The subnet references aws_vpc.main.id -- this is an implicit dependency
2. The internet gateway references the VPC ID -- another implicit dependency
3. The route table association references both the route table and the subnet
   
Answer these questions:

-  How does Terraform know to create the VPC before the subnet?
     1. Subnet is using VPC ID
     2. So subnet depends on VPC
    
- What would happen if you tried to create the subnet before the VPC existed?
    1.  Subnet needs a VPC
    2.  If VPC doesn’t exist → subnet cannot be created
  
- Find all implicit dependencies in your config and list them?
   
    1.  Subnet → VPC
         vpc_id = aws_vpc.main_vpc.id

    2. Internet Gateway → VPC
        vpc_id = aws_vpc.main_vpc.id

    3. Route Table → VPC
        vpc_id = aws_vpc.main_vpc.id

    4.  Route → Route Table + Internet Gateway
        route_table_id = aws_route_table.public_rt.id
        gateway_id     = aws_internet_gateway.igw.id

   5. Route Table Association → Subnet + Route Table
     subnet_id      = aws_subnet.public_subnet.id
      route_table_id = aws_route_table.public_rt.id

   6. Security Group → VPC
      vpc_id = aws_vpc.main_vpc.id

   7. EC2 Instance → Subnet + Security Group
       subnet_id              = aws_subnet.public_subnet.id
       vpc_security_group_ids = [aws_security_group.web_sg.id]

# Task 4: Add a Security Group and EC2 Instance

Add to your config:

1. aws_security_group in the VPC:

  - Ingress rule: allow SSH (port 22) from 0.0.0.0/0
  - Ingress rule: allow HTTP (port 80) from 0.0.0.0/0
  - Egress rule: allow all outbound traffic
  - Tag: "TerraWeek-SG"
    
2. aws_instance in the subnet:

  - Use Amazon Linux 2 AMI for your region
  - Instance type: t2.micro
  - Associate the security group
  - Set associate_public_ip_address = true
  - Tag: "TerraWeek-Server"
    
Apply and verify -- your EC2 instance should have a public IP and be reachable.

<img width="1920" height="1080" alt="Image" src="https://github.com/user-attachments/assets/11a7914a-cb54-43df-a934-f8f82177537f" />

<img width="919" height="241" alt="Image" src="https://github.com/user-attachments/assets/f1f1d7e0-7ac8-445d-ba59-6b5038328327" />

# Task 5: Explicit Dependencies with depends_on

Sometimes Terraform cannot detect a dependency automatically.

1. Add a second aws_s3_bucket resource for application logs
2. Add depends_on = [aws_instance.main] to the S3 bucket -- even though there is no direct reference, you want the bucket created only after the instance
3. Run terraform plan and observe the order

Now visualize the entire dependency tree:

    terraform graph | dot -Tpng > graph.png

If you don't have dot (Graphviz) installed, use:

     terraform graph
     
and paste the output into an online Graphviz viewer

<img width="1206" height="854" alt="Image" src="https://github.com/user-attachments/assets/8bc7e93e-34e5-4a53-a576-e46422c6a203" />

<img width="983" height="820" alt="Image" src="https://github.com/user-attachments/assets/b04c4bef-d0ce-4a52-a867-0ca048c9a36a" />

<img width="876" height="510" alt="Image" src="https://github.com/user-attachments/assets/0db2947d-f798-4eb5-bce2-5ba1aa4a32f2" />

<img width="478" height="534" alt="Image" src="https://github.com/user-attachments/assets/c2c9921e-591d-4cd3-aadc-bee76c3e3613" />
