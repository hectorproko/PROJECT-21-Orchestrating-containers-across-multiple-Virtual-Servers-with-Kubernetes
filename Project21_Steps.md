# PROJECT-21-Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes
Project 21

## KUBERNETES ARCHITECTURE  
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/PROJECT-21-Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes/main/images/K8s_architecture.png)

## INSTALL CLIENT TOOLS BEFORE BOOTSTRAPPING THE CLUSTER.

**Install and configure AWS CLI**  
Did not do anything here, planning to use AwS CLI configured already, Devops Account using Terraform User


**Installing kubectl**  
Did the wget
``` bash
hector@hector-Laptop:~$ chmod +x kubectl
hector@hector-Laptop:~$ ls -a | grep kubectl
kubectl
hector@hector-Laptop:~$ ls -l | grep kubectl
-rwxrwxr-x  1 hector hector 46436352 Apr  8  2021 kubectl
hector@hector-Laptop:~$ sudo mv kubectl /usr/local/bin/
[sudo] password for hector:
hector@hector-Laptop:~$ kubectl version --client
Client Version: version.Info{Major:"1", Minor:"21", GitVersion:"v1.21.0", GitCommit:"cb303e613a121a29364f75cc67d3d580833a7479", GitTreeState:"clean", BuildDate:"2021-04-08T16:31:21Z", GoVersion:"go1.16.1", Compiler:"gc", Platform:"linux/amd64"}
hector@hector-Laptop:~$
```

**Install CFSSL and CFSSLJSON**  
``` bash
hector@hector-Laptop:~$ wget -q --show-progress --https-only --timestamping \
>   https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssl \
>   https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssljson
cfssl               100%[===================>]  14.15M  2.53MB/s    in 5.7s
cfssljson           100%[===================>]   9.05M  1.99MB/s    in 5.2s
hector@hector-Laptop:~$ ls -l | grep cfssl
-rw-rw-r--  1 hector hector 14842064 Jul 18  2020 cfssl
-rw-rw-r--  1 hector hector  9495504 Jul 18  2020 cfssljson
hector@hector-Laptop:~$ chmod +x cfssl cfssljson
hector@hector-Laptop:~$ ls -l | grep cfssl
-rwxrwxr-x  1 hector hector 14842064 Jul 18  2020 cfssl
-rwxrwxr-x  1 hector hector  9495504 Jul 18  2020 cfssljson
hector@hector-Laptop:~$ sudo mv cfssl cfssljson /usr/local/bin/
[sudo] password for hector:
hector@hector-Laptop:~$ ls -l | grep cfssl
hector@hector-Laptop:~$
```


## AWS CLOUD RESOURCES FOR KUBERNETES CLUSTER  

`hector@hector-Laptop:~$ mkdir k8s-cluster-from-ground-up`

**Virtual Private Cloud – VPC**
``` bash
hector@hector-Laptop:~$ VPC_ID=$(aws ec2 create-vpc \
> --cidr-block 172.31.0.0/16 \
> --output text --query 'Vpc.VpcId'
> )
hector@hector-Laptop:~$ NAME=k8s-cluster-from-ground-up

hector@hector-Laptop:~$ aws ec2 create-tags \
>   --resources ${VPC_ID} \
>   --tags Key=Name,Value=${NAME}

hector@hector-Laptop:~$ aws ec2 modify-vpc-attribute \
> --vpc-id ${VPC_ID} \
> --enable-dns-support '{"Value": true}'

hector@hector-Laptop:~$ aws ec2 modify-vpc-attribute \
> --vpc-id ${VPC_ID} \
> --enable-dns-hostnames '{"Value": true}'
hector@hector-Laptop:~$
```

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/PROJECT-21-Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes/main/images/yourvpc.png)  

**Domain Name System – DNS**  
``` bash
hector@hector-Laptop:~$ DHCP_OPTION_SET_ID=$(aws ec2 create-dhcp-options \
>   --dhcp-configuration \
>     "Key=domain-name,Values=$AWS_REGION.hector.compute.internal" \
>     "Key=domain-name-servers,Values=AmazonProvidedDNS" \
>   --output text --query 'DhcpOptions.DhcpOptionsId')
hector@hector-Laptop:~$ #creates DHCP option set
hector@hector-Laptop:~$ aws ec2 create-tags \
>   --resources ${DHCP_OPTION_SET_ID} \
>   --tags Key=Name,Value=${NAME}
hector@hector-Laptop:~$ #adds the name to it, colum name
hector@hector-Laptop:~$
```
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/PROJECT-21-Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes/main/images/dhcp.png)  

`AWS_REGION=us-east-1`  

Associate the DHCP Option set with the VPC:  
``` bash
hector@hector-Laptop:~$ aws ec2 associate-dhcp-options \
>   --dhcp-options-id ${DHCP_OPTION_SET_ID} \
>   --vpc-id ${VPC_ID}
hector@hector-Laptop:~$
```

VPC > Your VPCs  
VPC is now associate with the above DHCP options set ID  
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/PROJECT-21-Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes/main/images/yourvpc2.png)  

Create the **Subnet**: 
``` bash
hector@hector-Laptop:~$ SUBNET_ID=$(aws ec2 create-subnet \
>   --vpc-id ${VPC_ID} \
>   --cidr-block 172.31.0.0/24 \
>   --output text --query 'Subnet.SubnetId')
aws ec2 create-tags \
  --resources ${SUBNET_ID} \
  --tags Key=Name,Value=${NAME}
hector@hector-Laptop:~$ aws ec2 create-tags \
>   --resources ${SUBNET_ID} \
>   --tags Key=Name,Value=${NAME}
hector@hector-Laptop:~$
```

VPC > Subnets  
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/PROJECT-21-Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes/main/images/subnets.png)  