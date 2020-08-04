# Hosting Wordpress Application, Introducing NAT gateway
In my previous article, we have seen how to launch the complete Wordpress application and also at the same time implementing security of the database by using some of the networking concepts like VPC, Subnets, Internet gateway, etc.

So, What exactly we did was, we created two subnets (public and private) and launched the front-end application in public world and MySql server in the private world which actually worked fine **but the problem we were facing was:**

1. If we don't have any public IP how to login to the system(MySQL instance).
2. How to upgrade software or anything for which we need internet access.
## Solution:
So the solution for our first problem is to use Wordpress instance as a **Bastion Host**. A **bastion host** is a server whose purpose is to provide access to a private network from an external network, such as the Internet.

And the solution for our second problem is NAT Gateway. **NAT Gateway** is a highly available AWS managed service that makes it easy to connect to the Internet from instances within a private subnet in an Amazon Virtual Private Cloud (Amazon VPC).

## Architecture:
![0](https://raw.githubusercontent.com/mohitagal98/hybrid-proj4/master/photos/vpc-ec2-diagram.png)
In short, if we need to manage the MySQL server we can logging it into it through the Wordpress's Instance.

And we will create a NAT gateway using which MySQL instance can access the Internet.
## Implementaion
In our previous article, we have already seen most of the implementation. So, in this article let's focus on adding another component i.e. NAT Gateway. So the flow is something like, first we need a Nat Gateway and then need to create a Route table which contains the information necessary to forward a packet along the Nat Gateway.

**_Create NAT gateway:_**
```
# Elastic IP
resource "aws_eip" "eip" {
depends_on=[
	aws_subnet.sn-pri-1b
]
  vpc      = true
}



# Nat Gateway
resource "aws_nat_gateway" "nat-gw" {
depends_on=[
	aws_eip.eip
]
  allocation_id = "${aws_eip.eip.id}"
  subnet_id     = "${aws_subnet.sn-pub-1a.id}"
}
```

For NAT gateway, first we need a public IP using resource **aws_eip** which we will assign to the NAT gateway. And put the NAT gateway into the public subnet as private subnet doesn't have outside connectivity.

**_Create Route Table:_**
```
# Route table for Private subnet
resource "aws_route_table" "mysql-route" {
depends_on=[
	aws_nat_gateway.nat-gw
]
  vpc_id = "${aws_vpc.app-vpc.id}"

  route {
    cidr_block = "0.0.0.0/0"
    nat_gateway_id = "${aws_nat_gateway.nat-gw.id}"
  }


  tags = {
    Name = "mysql-route"
  }
}
```
This will create a Route Table and at the same time providing Nat Gateway ID for the packets going to the public world. 

**Note**: As both the subnets have internal connectivity, so the packets from private subnet will first reach to NAT gateway which is in public subnet and then eventually to the public world.

**_Route_Table_Association:_**
```
#  route table association
resource "aws_route_table_association" "pri-association" {
depends_on=[
	aws_route_table.mysql-route
]
  subnet_id      = "${aws_subnet.sn-pri-1b.id}"
  route_table_id = "${aws_route_table.mysql-route.id}"
}
```

This will associate route table with private subnet.

**_Updating Security Group for MySQL:_**
```
# Security Group for mysql
resource "aws_security_group" "mysql-sg" {
depends_on=[
	aws_security_group.wp-sg
]
  name        = "mysql-sg"
  description = "Allows 3306 port."
  vpc_id      = "${aws_vpc.app-vpc.id}"

  ingress {
    from_port = 3306
    to_port   = 3306
    protocol  = "tcp"
    cidr_blocks = ["172.168.0.0/24"]
  }
  
  ingress {
    from_port = 22
    to_port   = 22
    protocol  = "tcp"
    cidr_blocks = ["172.168.0.0/24"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  tags = {
    Name = "allow mysql user"
  }
}
```

Updating security group by adding one ingress rule for port 22 which will allow to do SSH login into the MySQL instance from the Wordpress Instance.

All the necessary components are being added or updated now. To see the complete updated code [click here](https://github.com/mohitagal98/hybrid-proj4/blob/master/task4.tf).

### Finally, we are left with our two magical commands:
`terraform init`
`terraform apply -auto-approve`
Check resources:

Nat Gatway:
![1](https://raw.githubusercontent.com/mohitagal98/hybrid-proj4/master/photos/nat%20gateway.JPG)
Route tables:
![2](https://raw.githubusercontent.com/mohitagal98/hybrid-proj4/master/photos/route%20table.JPG)
Wordpress Blog:
![3](https://raw.githubusercontent.com/mohitagal98/hybrid-proj4/master/photos/blog.JPG)
MySQl access:
![4](https://raw.githubusercontent.com/mohitagal98/hybrid-proj4/master/photos/mysql%20login.JPG)
## Thank You !!
