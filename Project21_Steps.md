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

2. `kube-scheduler` **Client Certificate and Private Key**  

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


7. also
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




