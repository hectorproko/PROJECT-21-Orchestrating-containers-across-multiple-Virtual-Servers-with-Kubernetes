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


Create the **Internet Gateway** and attach it to the VPC:  

``` bash
hector@hector-Laptop:~$ INTERNET_GATEWAY_ID=$(aws ec2 create-internet-gateway \
>   --output text --query 'InternetGateway.InternetGatewayId')
aws ec2 create-tags \
  --resources ${INTERNET_GATEWAY_ID} \
  --tags Key=Name,Value=${NAME}
aws ec2 attach-internet-gateway \
  --internet-gateway-id ${INTERNET_GATEWAY_ID} \
  --vpc-id ${VPC_ID}
hector@hector-Laptop:~$ aws ec2 create-tags \
>   --resources ${INTERNET_GATEWAY_ID} \
>   --tags Key=Name,Value=${NAME}
hector@hector-Laptop:~$ aws ec2 attach-internet-gateway \
>   --internet-gateway-id ${INTERNET_GATEWAY_ID} \
>   --vpc-id ${VPC_ID}
hector@hector-Laptop:~$
```

VPC ID   
`vpc-003a8fe8a20274a1d`  

VPC > Internet gateways  
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/PROJECT-21-Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes/main/images/gateways.png)  

Create route tables, associate the **route table** to subnet, and create a route to allow external traffic to the Internet through the Internet Gateway:  

``` bash
hector@hector-Laptop:~$ ROUTE_TABLE_ID=$(aws ec2 create-route-table \
>   --vpc-id ${VPC_ID} \
>   --output text --query 'RouteTable.RouteTableId')
aws ec2 create-tags \
  --resources ${ROUTE_TABLE_ID} \
  --tags Key=Name,Value=${NAME}
aws ec2 associate-route-table \
  --route-table-id ${ROUTE_TABLE_ID} \
  --subnet-id ${SUBNET_ID}
aws ec2 create-route \
  --route-table-id ${ROUTE_TABLE_ID} \
  --destination-cidr-block 0.0.0.0/0 \
  --gateway-id ${INTERNET_GATEWAY_ID}
hector@hector-Laptop:~$ aws ec2 create-tags \
>   --resources ${ROUTE_TABLE_ID} \
>   --tags Key=Name,Value=${NAME}
hector@hector-Laptop:~$ aws ec2 associate-route-table \
>   --route-table-id ${ROUTE_TABLE_ID} \
>   --subnet-id ${SUBNET_ID}
{
    "AssociationId": "rtbassoc-0127a282e2bedeec0",
    "AssociationState": {
        "State": "associated"
    }
}
hector@hector-Laptop:~$ aws ec2 create-route \
>   --route-table-id ${ROUTE_TABLE_ID} \
>   --destination-cidr-block 0.0.0.0/0 \
>   --gateway-id ${INTERNET_GATEWAY_ID}
{
    "Return": true
}
hector@hector-Laptop:~$
```

VPC > Route tables
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/PROJECT-21-Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes/main/images/routetables.png)  



## SECURITY GROUPS  

**Security Groups**  
``` bash
hector@hector-Laptop:~$ # Create the security group and store its ID in a variable
hector@hector-Laptop:~$ SECURITY_GROUP_ID=$(aws ec2 create-security-group \
>   --group-name ${NAME} \
>   --description "Kubernetes cluster security group" \
>   --vpc-id ${VPC_ID} \
>   --output text --query 'GroupId')
# Create the NAME tag for the security group
aws ec2 create-tags \
  --resources ${SECURITY_GROUP_ID} \
  --tags Key=Name,Value=${NAME}
# Create Inbound traffic for all communication within the subnet to connect on ports used by the master node(s)
aws ec2 authorize-security-group-ingress \
    --group-id ${SECURITY_GROUP_ID} \
    --ip-permissions IpProtocol=tcp,FromPort=2379,ToPort=2380,IpRanges='[{CidrIp=172.31.0.0/24}]'
# # Create Inbound traffic for all communication within the subnet to connect on ports used by the worker nodes
aws ec2 authorize-security-group-ingress \
    --group-id ${SECURITY_GROUP_ID} \
    --ip-permissions IpProtocol=tcp,FromPort=30000,ToPort=32767,IpRanges='[{CidrIp=172.31.0.0/24}]'
# Create inbound traffic to allow connections to the Kubernetes API Server listening on port 6443
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol tcp \
  --port 6443 \
  --cidr 0.0.0.0/0
# Create Inbound traffic for SSH from anywhere (Do not do this in production. Limit access ONLY to IPs or CIDR that MUST connect)
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0
# Create ICMP ingress for all types
aws ec2 authorize-security-group-ingress \
  --group-id ${SECURITY_GROUP_ID} \
  --protocol icmp \
  --port -1 \
  --cidr 0.0.0.0/0
hector@hector-Laptop:~$ # Create the NAME tag for the security group
hector@hector-Laptop:~$ aws ec2 create-tags \
>   --resources ${SECURITY_GROUP_ID} \
>   --tags Key=Name,Value=${NAME}
hector@hector-Laptop:~$ # Create Inbound traffic for all communication within the subnet to connect on ports used by the master node(s)
hector@hector-Laptop:~$ aws ec2 authorize-security-group-ingress \
>     --group-id ${SECURITY_GROUP_ID} \
>     --ip-permissions IpProtocol=tcp,FromPort=2379,ToPort=2380,IpRanges='[{CidrIp=172.31.0.0/24}]'
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-0fdd63e60afe82980",
            "GroupId": "sg-0a9ee15e7bc3faf61",
            "GroupOwnerId": "199055125796",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 2379,
            "ToPort": 2380,
            "CidrIpv4": "172.31.0.0/24"
        }
    ]
}
hector@hector-Laptop:~$ # # Create Inbound traffic for all communication within the subnet to connect on ports used by the worker nodes
hector@hector-Laptop:~$ aws ec2 authorize-security-group-ingress \
>     --group-id ${SECURITY_GROUP_ID} \
>     --ip-permissions IpProtocol=tcp,FromPort=30000,ToPort=32767,IpRanges='[{CidrIp=172.31.0.0/24}]'
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-08c0785dd4c3d1add",
            "GroupId": "sg-0a9ee15e7bc3faf61",
            "GroupOwnerId": "199055125796",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 30000,
            "ToPort": 32767,
            "CidrIpv4": "172.31.0.0/24"
        }
    ]
}
hector@hector-Laptop:~$ # Create inbound traffic to allow connections to the Kubernetes API Server listening on port 6443
hector@hector-Laptop:~$ aws ec2 authorize-security-group-ingress \
>   --group-id ${SECURITY_GROUP_ID} \
>   --protocol tcp \
>   --port 6443 \
>   --cidr 0.0.0.0/0
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-0105bda399b098396",
            "GroupId": "sg-0a9ee15e7bc3faf61",
            "GroupOwnerId": "199055125796",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 6443,
            "ToPort": 6443,
            "CidrIpv4": "0.0.0.0/0"
        }
    ]
}
hector@hector-Laptop:~$ # Create Inbound traffic for SSH from anywhere (Do not do this in production. Limit access ONLY to IPs or CIDR that MUST connect)
hector@hector-Laptop:~$ aws ec2 authorize-security-group-ingress \
>   --group-id ${SECURITY_GROUP_ID} \
>   --protocol tcp \
>   --port 22 \
>   --cidr 0.0.0.0/0
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-0b646d465f5257b56",
            "GroupId": "sg-0a9ee15e7bc3faf61",
            "GroupOwnerId": "199055125796",
            "IsEgress": false,
            "IpProtocol": "tcp",
            "FromPort": 22,
            "ToPort": 22,
            "CidrIpv4": "0.0.0.0/0"
        }
    ]
}
hector@hector-Laptop:~$ # Create ICMP ingress for all types
hector@hector-Laptop:~$ aws ec2 authorize-security-group-ingress \
>   --group-id ${SECURITY_GROUP_ID} \
>   --protocol icmp \
>   --port -1 \
>   --cidr 0.0.0.0/0
{
    "Return": true,
    "SecurityGroupRules": [
        {
            "SecurityGroupRuleId": "sgr-0f3b453bf36f5ad8c",
            "GroupId": "sg-0a9ee15e7bc3faf61",
            "GroupOwnerId": "199055125796",
            "IsEgress": false,
            "IpProtocol": "icmp",
            "FromPort": -1,
            "ToPort": -1,
            "CidrIpv4": "0.0.0.0/0"
        }
    ]
}
hector@hector-Laptop:~$
```

VPC > Security > Security groups  
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/PROJECT-21-Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes/main/images/securitygroups.png)  


**Network Load Balancer**  
``` bash
hector@hector-Laptop:~$ LOAD_BALANCER_ARN=$(aws elbv2 create-load-balancer \
> --name ${NAME} \
> --subnets ${SUBNET_ID} \
> --scheme internet-facing \
> --type network \
> --output text --query 'LoadBalancers[].LoadBalancerArn')
hector@hector-Laptop:~$
```

EC2 > Load Balancing > Load Balancers  
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/PROJECT-21-Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes/main/images/createlb.png)  

**Target Group**  
``` bash
hector@hector-Laptop:~$ TARGET_GROUP_ARN=$(aws elbv2 create-target-group \
>   --name ${NAME} \
>   --protocol TCP \
>   --port 6443 \
>   --vpc-id ${VPC_ID} \
>   --target-type ip \
>   --output text --query 'TargetGroups[].TargetGroupArn')
hector@hector-Laptop:~$
```

EC2 > Load Balancing > Target Groups  
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/PROJECT-21-Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes/main/images/targetgroups.png)  

**Register targets**: (Just like above, no real targets. You will just put the IP addresses so that, when the nodes become available, they will be used as targets.)  

``` bash
hector@hector-Laptop:~$ aws elbv2 register-targets \
>   --target-group-arn ${TARGET_GROUP_ARN} \
>   --targets Id=172.31.0.1{0,1,2}
hector@hector-Laptop:~$
```

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/PROJECT-21-Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes/main/images/targetgroups2.png)  

Create a **listener** to listen for requests and forward to the target nodes on TCP port `6443`  

``` bash
hector@hector-Laptop:~$ aws elbv2 create-listener \
> --load-balancer-arn ${LOAD_BALANCER_ARN} \
> --protocol TCP \
> --port 6443 \
> --default-actions Type=forward,TargetGroupArn=${TARGET_GROUP_ARN} \
> --output text --query 'Listeners[].ListenerArn'
arn:aws:elasticloadbalancing:us-east-1:199055125796:listener/net/k8s-cluster-from-ground-up/a09ad605b1edac82/add00ef9131bd674
hector@hector-Laptop:~$
```

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/PROJECT-21-Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes/main/images/addlistener.png)  

**K8s Public Address**  

``` bash
hector@hector-Laptop:~$ KUBERNETES_PUBLIC_ADDRESS=$(aws elbv2 describe-load-balancers \
> --load-balancer-arns ${LOAD_BALANCER_ARN} \
> --output text --query 'LoadBalancers[].DNSName')
hector@hector-Laptop:~$

hector@hector-Laptop:~$ echo $KUBERNETES_PUBLIC_ADDRESS
k8s-cluster-from-ground-up-a09ad605b1edac82.elb.us-east-1.amazonaws.com
hector@hector-Laptop:~$
```




## CREATE COMPUTE RESOURCES

**AMI** (needed to install sudo apt install jq) (jq - Command-line JSON processor)  
``` bash
hector@hector-Laptop:~$ IMAGE_ID=$(aws ec2 describe-images --owners 099720109477 \
>   --filters \
>   'Name=root-device-type,Values=ebs' \
>   'Name=architecture,Values=x86_64' \
>   'Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-xenial-16.04-amd64-server-*' \
>   | jq -r '.Images|sort_by(.Name)[-1]|.ImageId')
hector@hector-Laptop:~$ echo $IMAGE_ID
ami-0b0ea68c435eb488d
hector@hector-Laptop:~$
```

**SSH key-pair**  
``` bash
hector@hector-Laptop:~$ mkdir -p ssh
hector@hector-Laptop:~$ aws ec2 create-key-pair \
>   --key-name ${NAME} \
>   --output text --query 'KeyMaterial' \
>   > ssh/${NAME}.id_rsa
chmod 600 ssh/${NAME}.id_rsa
hector@hector-Laptop:~$ chmod 600 ssh/${NAME}.id_rsa

hector@hector-Laptop:~$ ls ssh
k8s-cluster-from-ground-up.id_rsa
hector@hector-Laptop:~$
```

**EC2 Instances for Controle Plane (Master Nodes)**  
``` bash
hector@hector-Laptop:~$ for i in 0 1 2; do
>   instance_id=$(aws ec2 run-instances \
>     --associate-public-ip-address \
>     --image-id ${IMAGE_ID} \
>     --count 1 \
>     --key-name ${NAME} \
>     --security-group-ids ${SECURITY_GROUP_ID} \
>     --instance-type t2.micro \
>     --private-ip-address 172.31.0.1${i} \
>     --user-data "name=master-${i}" \
>     --subnet-id ${SUBNET_ID} \
>     --output text --query 'Instances[].InstanceId')
>   aws ec2 modify-instance-attribute \
>     --instance-id ${instance_id} \
>     --no-source-dest-check
>   aws ec2 create-tags \
>     --resources ${instance_id} \
>     --tags "Key=Name,Value=${NAME}-master-${i}"
> done
```

EC2 > Instances > Instances  
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/PROJECT-21-Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes/main/images/instances.png)  

**EC2 Instances for Worker Nodes**  
``` bash
hector@hector-Laptop:~$ for i in 0 1 2; do
>   instance_id=$(aws ec2 run-instances \
>     --associate-public-ip-address \
>     --image-id ${IMAGE_ID} \
>     --count 1 \
>     --key-name ${NAME} \
>     --security-group-ids ${SECURITY_GROUP_ID} \
>     --instance-type t2.micro \
>     --private-ip-address 172.31.0.2${i} \
>     --user-data "name=worker-${i}|pod-cidr=172.20.${i}.0/24" \
>     --subnet-id ${SUBNET_ID} \
>     --output text --query 'Instances[].InstanceId')
>   aws ec2 modify-instance-attribute \
>     --instance-id ${instance_id} \
>     --no-source-dest-check
>   aws ec2 create-tags \
>     --resources ${instance_id} \
>     --tags "Key=Name,Value=${NAME}-worker-${i}"
> done
hector@hector-Laptop:~$
```

![Markdown Logo](https://raw.githubusercontent.com/hectorproko/PROJECT-21-Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes/main/images/instances2.png)  

## PREPARE THE SELF-SIGNED CERTIFICATE AUTHORITY AND GENERATE TLS CERTIFICATES  

**Self-Signed Root Certificate Authority (CA)**  

`hector@hector-Laptop:~$ mkdir ca-authority && cd ca-authority`

``` bash
hector@hector-Laptop:~/ca-authority$ ls
hector@hector-Laptop:~/ca-authority$ {
> cat > ca-config.json <<EOF
> {
>   "signing": {
>     "default": {
>       "expiry": "8760h"
>     },
>     "profiles": {
>       "kubernetes": {
>         "usages": ["signing", "key encipherment", "server auth", "client auth"],
>         "expiry": "8760h"
>       }
>     }
>   }
> }
> EOF
> cat > ca-csr.json <<EOF
> {
>   "CN": "Kubernetes",
>   "key": {
>     "algo": "rsa",
>     "size": 2048
>   },
>   "names": [
>     {
>       "C": "US",
>       "L": "Florida",
>       "O": "Kubernetes",
>       "OU": "Hector DEVOPS",
>       "ST": "Miami"
>     }
>   ]
> }
> EOF
> cfssl gencert -initca ca-csr.json | cfssljson -bare ca
> }
2022/06/08 14:17:28 [INFO] generating a new CA key and certificate from CSR
2022/06/08 14:17:28 [INFO] generate received request
2022/06/08 14:17:28 [INFO] received CSR
2022/06/08 14:17:28 [INFO] generating key: rsa-2048
2022/06/08 14:17:28 [INFO] encoded CSR
2022/06/08 14:17:28 [INFO] signed certificate with serial number 595938750693050736385532716166856798624679790798
hector@hector-Laptop:~/ca-authority$
```


Generate the **Certificate Signing Request (CSR)**, **Private Key** and the **Certificate** for the Kubernetes Master Nodes.  

``` bash
hector@hector-Laptop:~/ca-authority$ {
> cat > master-kubernetes-csr.json <<EOF
> {
>   "CN": "kubernetes",
>    "hosts": [
>    "127.0.0.1",
>    "172.31.0.10",
>    "172.31.0.11",
>    "172.31.0.12",
>    "ip-172-31-0-10",
>    "ip-172-31-0-11",
>    "ip-172-31-0-12",
>    "ip-172-31-0-10.${AWS_REGION}.compute.internal",
>    "ip-172-31-0-11.${AWS_REGION}.compute.internal",
>    "ip-172-31-0-12.${AWS_REGION}.compute.internal",
>    "${KUBERNETES_PUBLIC_ADDRESS}",
>    "kubernetes",
>    "kubernetes.default",
>    "kubernetes.default.svc",
>    "kubernetes.default.svc.cluster",
>    "kubernetes.default.svc.cluster.local"
>   ],
>   "key": {
>     "algo": "rsa",
>     "size": 2048
>   },
>   "names": [
>     {
>       "C": "US",
>       "L": "Florida",
>       "O": "Kubernetes",
>       "OU": "Hector DEVOPS",
>       "ST": "Miami"
>     }
>   ]
> }
> EOF
> cfssl gencert \
>   -ca=ca.pem \
>   -ca-key=ca-key.pem \
>   -config=ca-config.json \
>   -profile=kubernetes \
>   master-kubernetes-csr.json | cfssljson -bare master-kubernetes
> }
2022/06/08 14:23:17 [INFO] generate received request
2022/06/08 14:23:17 [INFO] received CSR
2022/06/08 14:23:17 [INFO] generating key: rsa-2048
2022/06/08 14:23:17 [INFO] encoded CSR
2022/06/08 14:23:17 [INFO] signed certificate with serial number 547649748408320050103723158040590516113896364808
hector@hector-Laptop:~/ca-authority$
```

**Creating the other certificates: for the following Kubernetes components:**  

1. `kube-scheduler` **Client Certificate and Private Key**  

``` bash
hector@hector-Laptop:~/ca-authority$ {
> cat > kube-scheduler-csr.json <<EOF
> {
>   "CN": "system:kube-scheduler",
>   "key": {
>     "algo": "rsa",
>     "size": 2048
>   },
>   "names": [
>     {
>       "C": "US",
>       "L": "Florida",
>       "O": "system:kube-scheduler",
>       "OU": "Hector DEVOPS",
>       "ST": "Miami"
>     }
>   ]
> }
> EOF
> cfssl gencert \
>   -ca=ca.pem \
>   -ca-key=ca-key.pem \
>   -config=ca-config.json \
>   -profile=kubernetes \
>   kube-scheduler-csr.json | cfssljson -bare kube-scheduler
> }
2022/06/08 14:27:53 [INFO] generate received request
2022/06/08 14:27:53 [INFO] received CSR
2022/06/08 14:27:53 [INFO] generating key: rsa-2048
2022/06/08 14:27:53 [INFO] encoded CSR
2022/06/08 14:27:53 [INFO] signed certificate with serial number 622069761468098043960394959246888396290159189426
2022/06/08 14:27:53 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
hector@hector-Laptop:~/ca-authority$
```

3. `kube-proxy` **Client Certificate and Private Key**  

``` bash
hector@hector-Laptop:~/ca-authority$ cat > kube-proxy-csr.json <<EOF
> {
>   "CN": "system:kube-proxy",
>   "key": {
>     "algo": "rsa",
>     "size": 2048
>   },
>   "names": [
>     {
>       "C": "US",
>       "L": "Florida",
>       "O": "system:node-proxier",
>       "OU": "Hector DEVOPS",
>       "ST": "Miami"
>     }
>   ]
> }
> EOF
hector@hector-Laptop:~/ca-authority$ cfssl gencert \
>   -ca=ca.pem \
>   -ca-key=ca-key.pem \
>   -config=ca-config.json \
>   -profile=kubernetes \
>   kube-proxy-csr.json | cfssljson -bare kube-proxy
2022/06/08 14:45:22 [INFO] generate received request
2022/06/08 14:45:22 [INFO] received CSR
2022/06/08 14:45:22 [INFO] generating key: rsa-2048
}2022/06/08 14:45:22 [INFO] encoded CSR
2022/06/08 14:45:22 [INFO] signed certificate with serial number 176112969599687911110401190660797659018781968094
2022/06/08 14:45:22 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
```

4. `kube-controller-manager` **Client Certificate and Private Key**  
``` bash
hector@hector-Laptop:~/ca-authority$ {
> cat > kube-controller-manager-csr.json <<EOF
> {
>   "CN": "system:kube-controller-manager",
>   "key": {
>     "algo": "rsa",
>     "size": 2048
>   },
>   "names": [
>     {
>       "C": "US",
>       "L": "Florida",
>       "O": "system:kube-controller-manager",
>       "OU": "Hector DEVOPS",
>       "ST": "Miami"
>     }
>   ]
> }
> EOF
> cfssl gencert \
>   -ca=ca.pem \
>   -ca-key=ca-key.pem \
>   -config=ca-config.json \
>   -profile=kubernetes \
>   kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
> }
2022/06/08 15:50:39 [INFO] generate received request
2022/06/08 15:50:39 [INFO] received CSR
2022/06/08 15:50:39 [INFO] generating key: rsa-2048
2022/06/08 15:50:39 [INFO] encoded CSR
2022/06/08 15:50:39 [INFO] signed certificate with serial number 279440487414994410215011560698903695611018498249
2022/06/08 15:50:39 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
hector@hector-Laptop:~/ca-authority$
```

5. `kubelet` **Client Certificate and Private Key**  
   
``` bash
hector@hector-Laptop:~/ca-authority$ for i in 0 1 2; do
>   instance="${NAME}-worker-${i}"
>   instance_hostname="ip-172-31-0-2${i}"
>   cat > ${instance}-csr.json <<EOF
> {
>   "CN": "system:node:${instance_hostname}",
>   "key": {
>     "algo": "rsa",
>     "size": 2048
>   },
>   "names": [
>     {
>       "C": "US",
>       "L": "Florida",
>       "O": "system:nodes",
>       "OU": "Hector DEVOPS",
>       "ST": "Miami"
>     }
>   ]
> }
> EOF
> external_ip=$(aws ec2 describe-instances \
>     --filters "Name=tag:Name,Values=${instance}" \
>     --output text --query 'Reservations[].Instances[].PublicIpAddress')
> internal_ip=$(aws ec2 describe-instances \
>     --filters "Name=tag:Name,Values=${instance}" \
>     --output text --query 'Reservations[].Instances[].PrivateIpAddress')
> cfssl gencert \
>     -ca=ca.pem \
>     -ca-key=ca-key.pem \
>     -config=ca-config.json \
>     -hostname=${instance_hostname},${external_ip},${internal_ip} \
>     -profile=kubernetes \
>     ${NAME}-worker-${i}-csr.json | cfssljson -bare ${NAME}-worker-${i}
> done
2022/06/08 20:38:25 [INFO] generate received request
2022/06/08 20:38:25 [INFO] received CSR
2022/06/08 20:38:25 [INFO] generating key: rsa-2048
2022/06/08 20:38:25 [INFO] encoded CSR
2022/06/08 20:38:25 [INFO] signed certificate with serial number 88068063509947593848109566279871873559554569039
2022/06/08 20:38:27 [INFO] generate received request
2022/06/08 20:38:27 [INFO] received CSR
2022/06/08 20:38:27 [INFO] generating key: rsa-2048
2022/06/08 20:38:27 [INFO] encoded CSR
2022/06/08 20:38:27 [INFO] signed certificate with serial number 582231254599715575847880223126164829455969326988
2022/06/08 20:38:28 [INFO] generate received request
2022/06/08 20:38:28 [INFO] received CSR
2022/06/08 20:38:28 [INFO] generating key: rsa-2048
2022/06/08 20:38:30 [INFO] encoded CSR
2022/06/08 20:38:30 [INFO] signed certificate with serial number 205507639545489712146564463937190119075761906061
hector@hector-Laptop:~/ca-authority$
```


6. `kubernetes admin user's` **Client Certificate and Private Key**  
``` bash
   hector@hector-Laptop:~/ca-authority$ {
> cat > admin-csr.json <<EOF
> {
>   "CN": "admin",
>   "key": {
>     "algo": "rsa",
>     "size": 2048
>   },
>   "names": [
>     {
>       "C": "US",
>       "L": "Florida",
>       "O": "system:masters",
>       "OU": "Hector DEVOPS",
>       "ST": "Miami"
>     }
>   ]
> }
> EOF
> cfssl gencert \
>   -ca=ca.pem \
>   -ca-key=ca-key.pem \
>   -config=ca-config.json \
>   -profile=kubernetes \
>   admin-csr.json | cfssljson -bare admin
> }
2022/06/08 20:49:35 [INFO] generate received request
2022/06/08 20:49:35 [INFO] received CSR
2022/06/08 20:49:35 [INFO] generating key: rsa-2048
2022/06/08 20:49:35 [INFO] encoded CSR
2022/06/08 20:49:35 [INFO] signed certificate with serial number 140984123270054335235966735745247869171088770273
2022/06/08 20:49:35 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
hector@hector-Laptop:~/ca-authority$
   ```


7. In addition
``` bash
   hector@hector-Laptop:~/ca-authority$ {
> cat > service-account-csr.json <<EOF
> {
>   "CN": "service-accounts",
>   "key": {
>     "algo": "rsa",
>     "size": 2048
>   },
>   "names": [
>     {
>       "C": "US",
>       "L": "Florida",
>       "O": "Kubernetes",
>       "OU": "Hector DEVOPS",
>       "ST": "Miami"
>     }
>   ]
> }
> EOF
> cfssl gencert \
>   -ca=ca.pem \
>   -ca-key=ca-key.pem \
>   -config=ca-config.json \
>   -profile=kubernetes \
>   service-account-csr.json | cfssljson -bare service-account
> }
2022/06/08 21:09:08 [INFO] generate received request
2022/06/08 21:09:08 [INFO] received CSR
2022/06/08 21:09:08 [INFO] generating key: rsa-2048
2022/06/08 21:09:09 [INFO] encoded CSR
2022/06/08 21:09:09 [INFO] signed certificate with serial number 117554976092240248700077848849651368992033028376
2022/06/08 21:09:09 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").
hector@hector-Laptop:~/ca-authority$
```





## DISTRIBUTING THE CLIENT AND SERVER CERTIFICATES

**Distributing the Client and Server Certificates**  
``` bash
hector@hector-Laptop:~/ca-authority$ for i in 0 1 2; do
>   instance="${NAME}-worker-${i}"
>   external_ip=$(aws ec2 describe-instances \
>     --filters "Name=tag:Name,Values=${instance}" \
>     --output text --query 'Reservations[].Instances[].PublicIpAddress')
>   scp -i ../ssh/${NAME}.id_rsa \
>     ca.pem ${instance}-key.pem ${instance}.pem ubuntu@${external_ip}:~/; \
> done
The authenticity of host '3.90.65.208 (3.90.65.208)' can't be established.
ECDSA key fingerprint is SHA256:NR3IjdAA33E/5ZSy37qSl25w+Ei1uQuxBaakkuXnyX0.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '3.90.65.208' (ECDSA) to the list of known hosts.
ca.pem                                                                          100% 1342    25.8KB/s   00:00
k8s-cluster-from-ground-up-worker-0-key.pem                                     100% 1679    30.9KB/s   00:00
k8s-cluster-from-ground-up-worker-0.pem                                         100% 1505    25.5KB/s   00:00
The authenticity of host '34.227.92.141 (34.227.92.141)' can't be established.
ECDSA key fingerprint is SHA256:P3cCnXigFCnAzo4O00bEVHY5T11M7UPm25hbNMIBuC4.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '34.227.92.141' (ECDSA) to the list of known hosts.
ca.pem                                                                          100% 1342    23.0KB/s   00:00
k8s-cluster-from-ground-up-worker-1-key.pem                                     100% 1675    30.4KB/s   00:00
k8s-cluster-from-ground-up-worker-1.pem                                         100% 1505    24.1KB/s   00:00
The authenticity of host '100.25.137.116 (100.25.137.116)' can't be established.
ECDSA key fingerprint is SHA256:+G7SV9/P0jp0N6aDE8wZ0zOBKKiHazKipfFA7btq8Fo.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '100.25.137.116' (ECDSA) to the list of known hosts.
ca.pem                                                                          100% 1342    24.5KB/s   00:00
k8s-cluster-from-ground-up-worker-2-key.pem                                     100% 1675    28.7KB/s   00:00
k8s-cluster-from-ground-up-worker-2.pem                                         100% 1505    24.1KB/s   00:00
hector@hector-Laptop:~/ca-authority$

```

**Master or Controller node:** – Note that only the `api-server` related files will be sent over to the master nodes.  
``` bash
hector@hector-Laptop:~/ca-authority$ for i in 0 1 2; do
> instance="${NAME}-master-${i}" \
>   external_ip=$(aws ec2 describe-instances \
>     --filters "Name=tag:Name,Values=${instance}" \
>     --output text --query 'Reservations[].Instances[].PublicIpAddress')
>   scp -i ../ssh/${NAME}.id_rsa \
>     ca.pem ca-key.pem service-account-key.pem service-account.pem \
>     master-kubernetes.pem master-kubernetes-key.pem ubuntu@${external_ip}:~/;
> done
The authenticity of host '100.26.49.196 (100.26.49.196)' can't be established.
ECDSA key fingerprint is SHA256:/4rGAImFx/BZwNaqt1ykQGmzYIQXm5m7E5xCzwMD7F0.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '100.26.49.196' (ECDSA) to the list of known hosts.
ca.pem                                                                          100% 1342    14.8KB/s   00:00
ca-key.pem                                                                      100% 1679    19.7KB/s   00:00
service-account-key.pem                                                         100% 1675    16.0KB/s   00:00
service-account.pem                                                             100% 1432    19.8KB/s   00:00
master-kubernetes.pem                                                           100% 1862    18.7KB/s   00:00
master-kubernetes-key.pem                                                       100% 1679    20.0KB/s   00:00
The authenticity of host '54.210.195.212 (54.210.195.212)' can't be established.
ECDSA key fingerprint is SHA256:QGJXM4aYn5FUC6nEwy/ggnEJkPc1gCPW7Vtr2J7niAI.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '54.210.195.212' (ECDSA) to the list of known hosts.
ca.pem                                                                          100% 1342    17.6KB/s   00:00
ca-key.pem                                                                      100% 1679    18.0KB/s   00:00
service-account-key.pem                                                         100% 1675    12.6KB/s   00:00
service-account.pem                                                             100% 1432    24.0KB/s   00:00
master-kubernetes.pem                                                           100% 1862    23.1KB/s   00:00
master-kubernetes-key.pem                                                       100% 1679    26.5KB/s   00:00
The authenticity of host '54.237.87.176 (54.237.87.176)' can't be established.
ECDSA key fingerprint is SHA256:qgV+75d1s0XOw3YEfo5byFg8zo876/Fqm5rLddsZnzE.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '54.237.87.176' (ECDSA) to the list of known hosts.
ca.pem                                                                          100% 1342     8.9KB/s   00:00
ca-key.pem                                                                      100% 1679    17.1KB/s   00:00
service-account-key.pem                                                         100% 1675    13.7KB/s   00:00
service-account.pem                                                             100% 1432    10.5KB/s   00:00
master-kubernetes.pem                                                           100% 1862    16.2KB/s   00:00
master-kubernetes-key.pem                                                       100% 1679    27.3KB/s   00:00
hector@hector-Laptop:~/ca-authority$
```


## USE `KUBECTL` TO GENERATE KUBERNETES CONFIGURATION FILES FOR AUTHENTICATION

1. Generate the **kubelet** kubeconfig file  

``` bash
hector@hector-Laptop:~/ca-authority$ KUBERNETES_API_SERVER_ADDRESS=$(aws elbv2 describe-load-balancers --load-balancer-arns ${LOAD_BALANCER_ARN} --output text --query 'LoadBalancers[].DNSName')
hector@hector-Laptop:~/ca-authority$ for i in 0 1 2; do
> instance="${NAME}-worker-${i}"
> instance_hostname="ip-172-31-0-2${i}"
> # Set the kubernetes cluster in the kubeconfig file
>   kubectl config set-cluster ${NAME} \
>     --certificate-authority=ca.pem \
>     --embed-certs=true \
>     --server=https://$KUBERNETES_API_SERVER_ADDRESS:6443 \
>     --kubeconfig=${instance}.kubeconfig
> # Set the cluster credentials in the kubeconfig file
>   kubectl config set-credentials system:node:${instance_hostname} \
>     --client-certificate=${instance}.pem \
>     --client-key=${instance}-key.pem \
>     --embed-certs=true \
>     --kubeconfig=${instance}.kubeconfig
> # Set the context in the kubeconfig file
>   kubectl config set-context default \
>     --cluster=${NAME} \
>     --user=system:node:${instance_hostname} \
>     --kubeconfig=${instance}.kubeconfig
> kubectl config use-context default --kubeconfig=${instance}.kubeconfig
> done
Cluster "k8s-cluster-from-ground-up" set.
User "system:node:ip-172-31-0-20" set.
Context "default" created.
Switched to context "default".
Cluster "k8s-cluster-from-ground-up" set.
User "system:node:ip-172-31-0-21" set.
Context "default" created.
Switched to context "default".
Cluster "k8s-cluster-from-ground-up" set.
User "system:node:ip-172-31-0-22" set.
Context "default" created.
Switched to context "default".
hector@hector-Laptop:~/ca-authority$
```
``` bash
hector@hector-Laptop:~/ca-authority$ ls -ltr *.kubeconfig
-rw------- 1 hector hector 6511 Jun  8 21:25 k8s-cluster-from-ground-up-worker-0.kubeconfig
-rw------- 1 hector hector 6507 Jun  8 21:25 k8s-cluster-from-ground-up-worker-1.kubeconfig
-rw------- 1 hector hector 6507 Jun  8 21:25 k8s-cluster-from-ground-up-worker-2.kubeconfig
hector@hector-Laptop:~/ca-authority$
```

2. Generate the **kube-proxy** kubeconfig  

``` bash
hector@hector-Laptop:~/ca-authority$ {
>   kubectl config set-cluster ${NAME} \
>     --certificate-authority=ca.pem \
>     --embed-certs=true \
>     --server=https://${KUBERNETES_API_SERVER_ADDRESS}:6443 \
>     --kubeconfig=kube-proxy.kubeconfig
> kubectl config set-credentials system:kube-proxy \
>     --client-certificate=kube-proxy.pem \
>     --client-key=kube-proxy-key.pem \
>     --embed-certs=true \
>     --kubeconfig=kube-proxy.kubeconfig
> kubectl config set-context default \
>     --cluster=${NAME} \
>     --user=system:kube-proxy \
>     --kubeconfig=kube-proxy.kubeconfig
> kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
> }
Cluster "k8s-cluster-from-ground-up" set.
User "system:kube-proxy" set.
Context "default" created.
Switched to context "default".
hector@hector-Laptop:~/ca-authority$
```

3. Generate the **Kube-Controller-Manager** kubeconfig  

``` bash
hector@hector-Laptop:~/ca-authority$ {
>   kubectl config set-cluster ${NAME} \
>     --certificate-authority=ca.pem \
>     --embed-certs=true \
>     --server=https://127.0.0.1:6443 \
>     --kubeconfig=kube-controller-manager.kubeconfig
> kubectl config set-credentials system:kube-controller-manager \
>     --client-certificate=kube-controller-manager.pem \
>     --client-key=kube-controller-manager-key.pem \
>     --embed-certs=true \
>     --kubeconfig=kube-controller-manager.kubeconfig
> kubectl config set-context default \
>     --cluster=${NAME} \
>     --user=system:kube-controller-manager \
>     --kubeconfig=kube-controller-manager.kubeconfig
> kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
> }
Cluster "k8s-cluster-from-ground-up" set.
User "system:kube-controller-manager" set.
Context "default" created.
Switched to context "default".
hector@hector-Laptop:~/ca-authority$
```

4. Generating the **Kube-Scheduler** Kubeconfig  

``` bash
hector@hector-Laptop:~/ca-authority$ {
> kubectl config set-cluster ${NAME} \
    --certificate-authority=ca.pem \
>     --certificate-authority=ca.pem \
>     --embed-certs=true \
>     --server=https://127.0.0.1:6443 \
>     --kubeconfig=kube-scheduler.kubeconfig
> kubectl config set-credentials system:kube-scheduler \
>     --client-certificate=kube-scheduler.pem \
>     --client-key=kube-scheduler-key.pem \
>     --embed-certs=true \
>     --kubeconfig=kube-scheduler.kubeconfig
> kubectl config set-context default \
>     --cluster=${NAME} \
>     --user=system:kube-scheduler \
>     --kubeconfig=kube-scheduler.kubeconfig
> kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
> }
Cluster "k8s-cluster-from-ground-up" set.
User "system:kube-scheduler" set.
Context "default" created.
Switched to context "default".
hector@hector-Laptop:~/ca-authority$
```

5. Finally, generate the kubeconfig file for the **admin user**

``` bash
hector@hector-Laptop:~/ca-authority$ {
>   kubectl config set-cluster ${NAME} \
>     --certificate-authority=ca.pem \
>     --embed-certs=true \
>     --server=https://${KUBERNETES_API_SERVER_ADDRESS}:6443 \
>     --kubeconfig=admin.kubeconfig
> kubectl config set-credentials admin \
>     --client-certificate=admin.pem \
>     --client-key=admin-key.pem \
>     --embed-certs=true \
>     --kubeconfig=admin.kubeconfig
> kubectl config set-context default \
>     --cluster=${NAME} \
>     --user=admin \
>     --kubeconfig=admin.kubeconfig
> kubectl config use-context default --kubeconfig=admin.kubeconfig
> }
Cluster "k8s-cluster-from-ground-up" set.
User "admin" set.
Context "default" created.
Switched to context "default".
hector@hector-Laptop:~/ca-authority$
```

Distribute the files to their respective servers, using `scp` and a for loop  

**Worker**  
``` bash
hector@hector-Laptop:~/ca-authority$ for i in 0 1 2; do
>   instance="${NAME}-worker-${i}"
>   external_ip=$(aws ec2 describe-instances \
>     --filters "Name=tag:Name,Values=${instance}" \
>     --output text --query 'Reservations[].Instances[].PublicIpAddress')
>   scp -i ../ssh/${NAME}.id_rsa \
>   ${instance}.kubeconfig kube-proxy.kubeconfig ubuntu@${external_ip}:~/; \
> done
k8s-cluster-from-ground-up-worker-0.kubeconfig                                                              100% 6511   101.2KB/s   00:00
kube-proxy.kubeconfig                                                                                       100% 6342   100.6KB/s   00:00
k8s-cluster-from-ground-up-worker-1.kubeconfig                                                              100% 6507   102.2KB/s   00:00
kube-proxy.kubeconfig                                                                                       100% 6342   103.5KB/s   00:00
k8s-cluster-from-ground-up-worker-2.kubeconfig                                                              100% 6507   105.6KB/s   00:00
kube-proxy.kubeconfig                                                                                       100% 6342    96.1KB/s   00:00
hector@hector-Laptop:~/ca-authority$

```
**Master**  
``` bash
hector@hector-Laptop:~/ca-authority$ for i in 0 1 2; do
> instance="${NAME}-master-${i}" \
>   external_ip=$(aws ec2 describe-instances \
>     --filters "Name=tag:Name,Values=${instance}" \
>     --output text --query 'Reservations[].Instances[].PublicIpAddress')
>   scp -i ../ssh/${NAME}.id_rsa \
>   kube-controller-manager.kubeconfig kube-scheduler.kubeconfig ubuntu@${external_ip}:~/;
> done
kube-controller-manager.kubeconfig                                                                          100% 6425   104.3KB/s   00:00
kube-scheduler.kubeconfig                                                                                   100% 6371    89.5KB/s   00:00
kube-controller-manager.kubeconfig                                                                          100% 6425    87.8KB/s   00:00
kube-scheduler.kubeconfig                                                                                   100% 6371    99.5KB/s   00:00
kube-controller-manager.kubeconfig                                                                          100% 6425   102.6KB/s   00:00
kube-scheduler.kubeconfig                                                                                   100% 6371    86.1KB/s   00:00
hector@hector-Laptop:~/ca-authority$
```

## PREPARE THE ETCD DATABASE FOR ENCRYPTION AT REST
``` bash
hector@hector-Laptop:~/ca-authority$ ETCD_ENCRYPTION_KEY=$(head -c 64 /dev/urandom | base64)
hector@hector-Laptop:~/ca-authority$ echo $ETCD_ENCRYPTION_KEY
ibbYlKxF8d9rfVJrNB3qGuSvw8JPtNw1dnEOHpYxSppc2uRz91buFt9iF1VTYgSkXnlW73y9dReR saUXK3gDVw==
hector@hector-Laptop:~/ca-authority$
```

Create an `encryption-config.yaml` file as documented officially by kubernetes

``` bash
hector@hector-Laptop:~/ca-authority$ cat > encryption-config.yaml <<EOF
> kind: EncryptionConfig
> apiVersion: v1
> resources:
>   - resources:
>       - secrets
>     providers:
>       - aescbc:
>           keys:
>             - name: key1
>               secret: ${ETCD_ENCRYPTION_KEY}
>       - identity: {}
> EOF
hector@hector-Laptop:~/ca-authority$ ls | grep encryption
encryption-config.yaml
```

``` bash
hector@hector-Laptop:~/ca-authority$ bat encryption-config.yaml
───────┬──────────────────────────────────────────────────────────────────────────────────────
       │ File: encryption-config.yaml
───────┼──────────────────────────────────────────────────────────────────────────────────────
   1   │ kind: EncryptionConfig
   2   │ apiVersion: v1
   3   │ resources:
   4   │   - resources:
   5   │       - secrets
   6   │     providers:
   7   │       - aescbc:
   8   │           keys:
   9   │             - name: key1
  10   │               secret: ibbYlKxF8d9rfVJrNB3qGuSvw8JPtNw1dnEOHpYxSppc2uRz91buFt9iF1VTYgSkXnlW73y9dReR
  11   │ saUXK3gDVw==
  12   │       - identity: {}
───────┴────────────────────────────────────────────────────────────────────────────────────────
hector@hector-Laptop:~/ca-authority$
```

Send the encryption file to the **Controller nodes** using `scp` and a `for` loop.  

**Master**:
``` bash
hector@hector-Laptop:~/ca-authority$ for i in 0 1 2; do
> instance="${NAME}-master-${i}" \
>   external_ip=$(aws ec2 describe-instances \
>     --filters "Name=tag:Name,Values=${instance}" \
>     --output text --query 'Reservations[].Instances[].PublicIpAddress')
>   scp -i ../ssh/${NAME}.id_rsa \
>   encryption-config.yaml ubuntu@${external_ip}:~/;
> done
encryption-config.yaml                                                                                      100%  285     4.9KB/s   00:00
encryption-config.yaml                                                                                      100%  285     5.7KB/s   00:00
encryption-config.yaml                                                                                      100%  285     5.6KB/s   00:00
hector@hector-Laptop:~/ca-authority$
```


SSH into the controller server  
``` bash
hector@hector-Laptop:~/ca-authority$ cd ~/.ssh/
hector@hector-Laptop:~/.ssh$
```

The **.pem** key we need k8s-cluster-from-ground-up.id_rsa is in a **ssh** folder in home directory  
``` bash
hector@hector-Laptop:~/ssh$ pwd
/home/hector/ssh
hector@hector-Laptop:~/ssh$ ls
k8s-cluster-from-ground-up.id_rsa
hector@hector-Laptop:~/ssh$
```

1. SSH into the controller server

**Master1** (after logged in checking files)
``` bash
ubuntu@ip-172-31-0-10:~$ ls -l
total 44
-rw------- 1 ubuntu ubuntu 1679 Jun  9 01:17 ca-key.pem
-rw-rw-r-- 1 ubuntu ubuntu 1342 Jun  9 01:17 ca.pem
-rw-rw-r-- 1 ubuntu ubuntu  285 Jun 15 15:41 encryption-config.yaml
-rw------- 1 ubuntu ubuntu 6425 Jun 15 14:52 kube-controller-manager.kubeconfig
-rw------- 1 ubuntu ubuntu 6371 Jun 15 14:52 kube-scheduler.kubeconfig
-rw------- 1 ubuntu ubuntu 1679 Jun  9 01:17 master-kubernetes-key.pem
-rw-rw-r-- 1 ubuntu ubuntu 1862 Jun  9 01:17 master-kubernetes.pem
-rw------- 1 ubuntu ubuntu 1675 Jun  9 01:17 service-account-key.pem
-rw-rw-r-- 1 ubuntu ubuntu 1432 Jun  9 01:17 service-account.pem
ubuntu@ip-172-31-0-10:~$
```

**Master2**
``` bash
ubuntu@ip-172-31-0-11:~$ ls -l
total 44
-rw------- 1 ubuntu ubuntu 1679 Jun  9 01:17 ca-key.pem
-rw-rw-r-- 1 ubuntu ubuntu 1342 Jun  9 01:17 ca.pem
-rw-rw-r-- 1 ubuntu ubuntu  285 Jun 15 15:41 encryption-config.yaml
-rw------- 1 ubuntu ubuntu 6425 Jun 15 14:52 kube-controller-manager.kubeconfig
-rw------- 1 ubuntu ubuntu 6371 Jun 15 14:52 kube-scheduler.kubeconfig
-rw------- 1 ubuntu ubuntu 1679 Jun  9 01:17 master-kubernetes-key.pem
-rw-rw-r-- 1 ubuntu ubuntu 1862 Jun  9 01:17 master-kubernetes.pem
-rw------- 1 ubuntu ubuntu 1675 Jun  9 01:17 service-account-key.pem
-rw-rw-r-- 1 ubuntu ubuntu 1432 Jun  9 01:17 service-account.pem
ubuntu@ip-172-31-0-11:~$
```

**Master3**
``` bash
ubuntu@ip-172-31-0-12:~$ ls -l
total 44
-rw------- 1 ubuntu ubuntu 1679 Jun  9 01:17 ca-key.pem
-rw-rw-r-- 1 ubuntu ubuntu 1342 Jun  9 01:17 ca.pem
-rw-rw-r-- 1 ubuntu ubuntu  285 Jun 15 15:41 encryption-config.yaml
-rw------- 1 ubuntu ubuntu 6425 Jun 15 14:52 kube-controller-manager.kubeconfig
-rw------- 1 ubuntu ubuntu 6371 Jun 15 14:52 kube-scheduler.kubeconfig
-rw------- 1 ubuntu ubuntu 1679 Jun  9 01:17 master-kubernetes-key.pem
-rw-rw-r-- 1 ubuntu ubuntu 1862 Jun  9 01:17 master-kubernetes.pem
-rw------- 1 ubuntu ubuntu 1675 Jun  9 01:17 service-account-key.pem
-rw-rw-r-- 1 ubuntu ubuntu 1432 Jun  9 01:17 service-account.pem
ubuntu@ip-172-31-0-12:~$
```


2. Download and install **etcd**  
``` bash
ubuntu@ip-172-31-0-10:~$ wget -q --show-progress --https-only --timestamping \
>   " https://github.com/etcd-io/etcd/releases/download/v3.4.15/etcd-v3.4.15-linux-amd64.tar.gz"
etcd-v3.4.15-linux-amd64.tar.gz                  100%[========================================================================================================>]  16.60M   108MB/s    in 0.2s
ubuntu@ip-172-31-0-10:~$
```

3. Extract and install the etcd server and the etcdctl command line utility: ( 3 master instances done)
`tar -xvf etcd-v3.4.15-linux-amd64.tar.gz && sudo mv etcd-v3.4.15-linux-amd64/etcd* /usr/local/bin/`
``` bash
ubuntu@ip-172-31-0-10:~/etcd-v3.4.15-linux-amd64$ ls /usr/local/bin/
etcd  etcdctl
```
4. Configure the **etcd** server (3 master instances done)
``` bash
ubuntu@ip-172-31-0-10:~$ {
>   sudo mkdir -p /etc/etcd /var/lib/etcd
>   sudo chmod 700 /var/lib/etcd
>   sudo cp ca.pem master-kubernetes-key.pem master-kubernetes.pem /etc/etcd/
> }
sudo: unable to resolve host ip-172-31-0-10
sudo: unable to resolve host ip-172-31-0-10
sudo: unable to resolve host ip-172-31-0-10
ubuntu@ip-172-31-0-10:~$ ls /etc/etcd/
ca.pem  master-kubernetes-key.pem  master-kubernetes.pem
```

5. The instance internal IP address will be used to serve client requests and communicate with **etcd** cluster peers. Retrieve the internal IP address for the current compute instance:
`export INTERNAL_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)`

6. Each **etcd** member must have a unique name within an **etcd** cluster. Set the **etcd** name to node Private IP address so it will uniquely identify the machine:

``` bash
ubuntu@ip-172-31-0-11:~$ ETCD_NAME=$(curl -s http://169.254.169.254/latest/user-data/ \
>   | tr "|" "\n" | grep "^name" | cut -d"=" -f2)
ubuntu@ip-172-31-0-11:~$ echo ${ETCD_NAME}
master-1 #example of one done
ubuntu@ip-172-31-0-11:~$
```


7. Create the etcd.service systemd unit file: (in 3 masters)

``` bash
ubuntu@ip-172-31-0-10:/etc/systemd/system$ ls | grep etcd
etcd.service
ubuntu@ip-172-31-0-10:/etc/systemd/system$ cat etcd.service
[Unit]
Description=etcd
Documentation=https://github.com/coreos
[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \
  --name master-0 \
  --trusted-ca-file=/etc/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ca.pem \
  --peer-client-cert-auth \
  --client-cert-auth \
  --listen-peer-urls https://172.31.0.10:2380 \
  --listen-client-urls https://172.31.0.10:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://172.31.0.10:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster master-0=https://172.31.0.10:2380,master-1=https://172.31.0.11:2380,master-2=https://172.31.0.12:2380 \
  --cert-file=/etc/etcd/master-kubernetes.pem \
  --key-file=/etc/etcd/master-kubernetes-key.pem \
  --peer-cert-file=/etc/etcd/master-kubernetes.pem \
  --peer-key-file=/etc/etcd/master-kubernetes-key.al-advertise-peepem \
  --initir-urls https://0 \
  --initia{INTERNAL_IP}:23l-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
ubuntu@ip-172-31-0-10:/etc/systemd/system$
```


8. Start and enable the **etcd** Server
``` bash
ubuntu@ip-172-31-0-12:~$ {
> sudo systemctl daemon-reload
> sudo systemctl enable etcd
> sudo systemctl start etcd
> }
sudo: unable to resolve host ip-172-31-0-12
sudo: unable to resolve host ip-172-31-0-12
sudo: unable to resolve host ip-172-31-0-12
ubuntu@ip-172-31-0-12:~$
```
**Now all 3 master have status active**  
![Markdown Logo](https://raw.githubusercontent.com/hectorproko/PROJECT-21-Orchestrating-containers-across-multiple-Virtual-Servers-with-Kubernetes/main/images/activerunning.png)  

9. Verify the **etcd** installation  

``` bash
ubuntu@ip-172-31-0-10:~$ sudo ETCDCTL_API=3 etcdctl member list \
>   --endpoints=https://127.0.0.1:2379 \
>   --cacert=/etc/etcd/ca.pem \
>   --cert=/etc/etcd/master-kubernetes.pem \
>   --key=/etc/etcd/master-kubernetes-key.pem
sudo: unable to resolve host ip-172-31-0-10
6709c481b5234095, started, master-0, https://172.31.0.10:2380, https://172.31.0.10:2379, false
ade74a4f39c39f33, started, master-1, https://172.31.0.11:2380, https://172.31.0.11:2379, false
ed33b44c0b153ee3, started, master-2, https://172.31.0.12:2380, https://172.31.0.12:2379, false
ubuntu@ip-172-31-0-10:~$
```
``` bash
ubuntu@ip-172-31-0-11:~$ sudo ETCDCTL_API=3 etcdctl member list \
>   --endpoints=https://127.0.0.1:2379 \
>   --cacert=/etc/etcd/ca.pem \
>   --cert=/etc/etcd/master-kubernetes.pem \
>   --key=/etc/etcd/master-kubernetes-key.pem
sudo: unable to resolve host ip-172-31-0-11
6709c481b5234095, started, master-0, https://172.31.0.10:2380, https://172.31.0.10:2379, false
ade74a4f39c39f33, started, master-1, https://172.31.0.11:2380, https://172.31.0.11:2379, false
ed33b44c0b153ee3, started, master-2, https://172.31.0.12:2380, https://172.31.0.12:2379, false
ubuntu@ip-172-31-0-11:~$
```
``` bash
ubuntu@ip-172-31-0-12:~$ sudo ETCDCTL_API=3 etcdctl member list \
>   --endpoints=https://127.0.0.1:2379 \
>   --cacert=/etc/etcd/ca.pem \
>   --cert=/etc/etcd/master-kubernetes.pem \
>   --key=/etc/etcd/master-kubernetes-key.pem
sudo: unable to resolve host ip-172-31-0-12
6709c481b5234095, started, master-0, https://172.31.0.10:2380, https://172.31.0.10:2379, false
ade74a4f39c39f33, started, master-1, https://172.31.0.11:2380, https://172.31.0.11:2379, false
ed33b44c0b153ee3, started, master-2, https://172.31.0.12:2380, https://172.31.0.12:2379, false
ubuntu@ip-172-31-0-12:~$
```



## BOOTSTRAP THE CONTROL PLANE

1. Create the Kubernetes configuration directory:  
`sudo mkdir -p /etc/kubernetes/config`

2. Download the official Kubernetes release binaries:  
``` bash
ubuntu@ip-172-31-0-10:~$ wget -q --show-progress --https-only --timestamping \
> "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-apiserver" \
> "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-controller-manager" \
> "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kube-scheduler" \
> "https://storage.googleapis.com/kubernetes-release/release/v1.21.0/bin/linux/amd64/kubectl"
kube-apiserver                      100%[==================================================================>] 116.41M  94.1MB/s    in 1.2s
kube-controller-manager             100%[==================================================================>] 110.89M   100MB/s    in 1.1s
kube-scheduler                      100%[==================================================================>]  44.92M  90.3MB/s    in 0.5s
kubectl                             100%[==================================================================>]  44.29M  98.6MB/s    in 0.4s
ubuntu@ip-172-31-0-10:~$
```

3. Install the Kubernetes binaries:  
``` bash
ubuntu@ip-172-31-0-10:~$ {
> chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl
> sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/
> }
sudo: unable to resolve host ip-172-31-0-10
ubuntu@ip-172-31-0-10:~$ ls /usr/local/bin/
etcd  etcdctl  kube-apiserver  kube-controller-manager  kubectl  kube-scheduler
ubuntu@ip-172-31-0-10:~$
```

4. Configure the Kubernetes API Server:  
``` bash
ubuntu@ip-172-31-0-10:~$ {
> sudo mkdir -p /var/lib/kubernetes/
>
> sudo mv ca.pem ca-key.pem master-kubernetes-key.pem master-kubernetes.pem \
> service-account-key.pem service-account.pem \
> encryption-config.yaml /var/lib/kubernetes/
> }
sudo: unable to resolve host ip-172-31-0-10
sudo: unable to resolve host ip-172-31-0-10
ubuntu@ip-172-31-0-10:~$ ls /var/lib/kubernetes/
ca-key.pem  ca.pem  encryption-config.yaml  master-kubernetes-key.pem  master-kubernetes.pem  service-account-key.pem  service-account.pem
ubuntu@ip-172-31-0-10:~$
```

Get internal IP to build file  
`export INTERNAL_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)`  

5. Configure the Kubernetes Controller Manager:  

Move the kube-controller-manager kubeconfig into place:  
`sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/`

``` bash
ubuntu@ip-172-31-0-10:~$ sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
sudo: unable to resolve host ip-172-31-0-10
ubuntu@ip-172-31-0-10:~$ ls /var/lib/kubernetes/
ca-key.pem  encryption-config.yaml              master-kubernetes-key.pem  service-account-key.pem
ca.pem      kube-controller-manager.kubeconfig  master-kubernetes.pem      service-account.pem
ubuntu@ip-172-31-0-10:~$
```

Export some variables to retrieve the `vpc_cidr` – This will be required for the bind-address flag:  
``` bash
ubuntu@ip-172-31-0-12:~$ export AWS_METADATA="http://169.254.169.254/latest/meta-data"
ubuntu@ip-172-31-0-12:~$ export EC2_MAC_ADDRESS=$(curl -s $AWS_METADATA/network/interfaces/macs/ | head -n1 | tr -d '/')
ubuntu@ip-172-31-0-12:~$ export VPC_CIDR=$(curl -s $AWS_METADATA/network/interfaces/macs/$EC2_MAC_ADDRESS/vpc-ipv4-cidr-block/)
ubuntu@ip-172-31-0-12:~$ export NAME=k8s-cluster-from-ground-up
ubuntu@ip-172-31-0-12:~$ echo $AWS_METADATA
http://169.254.169.254/latest/meta-data
ubuntu@ip-172-31-0-12:~$ echo $EC2_MAC_ADDRESS
06:c9:ba:4b:28:03
ubuntu@ip-172-31-0-12:~$ echo $VPC_CIDR
172.31.0.0/16
ubuntu@ip-172-31-0-12:~$ echo $NAME
k8s-cluster-from-ground-up
ubuntu@ip-172-31-0-12:~$
```
Create the kube-controller-manager.service systemd unit file:  
``` bash
ubuntu@ip-172-31-0-10:~$ ls /etc/systemd/system/ | grep kube-controller-manager.service
kube-controller-manager.service
ubuntu@ip-172-31-0-10:~$
```

6. Configure the Kubernetes Scheduler:

Move the kube-scheduler kubeconfig into place:
``` bash
ubuntu@ip-172-31-0-10:~$ sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
sudo: unable to resolve host ip-172-31-0-10
ubuntu@ip-172-31-0-10:~$ sudo mkdir -p /etc/kubernetes/config
sudo: unable to resolve host ip-172-31-0-10
ubuntu@ip-172-31-0-10:~$ ls /var/lib/kubernetes/ | grep kube-schedul
kube-scheduler.kubeconfig
ubuntu@ip-172-31-0-10:~$ ls /etc/kubernetes/
config
ubuntu@ip-172-31-0-10:~$
```

Create the `kube-scheduler.yaml` configuration file:  
``` bash
ubuntu@ip-172-31-0-10:~$ cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
> apiVersion: kubescheduler.config.k8s.io/v1beta1
> kind: KubeSchedulerConfiguration
> clientConnection:
>   kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
> leaderElection:
>   leaderElect: true
> EOF
sudo: unable to resolve host ip-172-31-0-10
apiVersion: kubescheduler.config.k8s.io/v1beta1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
ubuntu@ip-172-31-0-10:~$
```

Create the kube-scheduler.service systemd unit file:  

``` bash
ubuntu@ip-172-31-0-10:~$ cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
> [Unit]
> Description=Kubernetes Scheduler
> Documentation=https://github.com/kubernetes/kubernetes
> [Service]
> ExecStart=/usr/local/bin/kube-scheduler \\
>   --config=/etc/kubernetes/config/kube-scheduler.yaml \\
>   --v=2
> Restart=on-failure
> RestartSec=5
> [Install]
> WantedBy=multi-user.target
> EOF
sudo: unable to resolve host ip-172-31-0-10
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
[Service]
ExecStart=/usr/local/bin/kube-scheduler \
  --config=/etc/kubernetes/config/kube-scheduler.yaml \
  --v=2
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
ubuntu@ip-172-31-0-10:~$
```


7. Start the Controller Services

