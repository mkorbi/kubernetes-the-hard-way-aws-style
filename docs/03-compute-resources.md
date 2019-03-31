# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across a single region.

> Ensure a default region has been set as described in the [Prerequisites](01-prerequisites.md#set-a-default-compute-region-and-zone) lab.

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Private Cloud Network - VPC

In this section a dedicated [Virtual Private Cloud](https://aws.amazon.com/vpc/) (VPC) network will be setup to host the Kubernetes cluster.

Create the VPC network:

```
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.240.0.0/24 --output text --query 'Vpc.VpcId')
aws ec2 create-tags --resources ${VPC_ID} --tags Key=Name,Value=kubernetes-the-hard-way
aws ec2 modify-vpc-attribute --vpc-id ${VPC_ID} --enable-dns-support '{"Value": true}'
aws ec2 modify-vpc-attribute --vpc-id ${VPC_ID} --enable-dns-hostnames '{"Value": true}'
```
Remember the VPC ID which you get as output from the command above. The VPC ID starts with "vpc-" and some alphanumeric characters.

Beside the VPC we need to create a subnet.
```
SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id ${VPC_ID} \
  --cidr-block 10.240.0.0/24 \
  --output text --query 'Subnet.SubnetId')
aws ec2 create-tags --resources ${SUBNET_ID} --tags Key=Name,Value=k8s
```

### Internet Gateway
Next, make your VPC accessable from the internet. For this you need to create a Internet Gateway, which you then attach to your VPC.
```
INTERNET_GATEWAY_ID=$(aws ec2 create-internet-gateway --output text --query 'InternetGateway.InternetGatewayId')
aws ec2 create-tags --resources ${INTERNET_GATEWAY_ID} --tags Key=Name,Value=k8s
```
And attach the internet gateway to your VPC
```
aws ec2 attach-internet-gateway --vpc-id ${VPC_ID}--internet-gateway-id ${INTERNET_GATEWAY_ID}
```

### Security Groups
To protect your instances and network at least a little, we will create a Security Group (sg).

SECURITY_GROUP_ID=$(aws ec2 create-security-group \
  --group-name k8s \
  --description "Kubernetes security group" \
  --vpc-id ${VPC_ID} \
  --output text --query 'GroupId')
aws ec2 create-tags --resources ${SECURITY_GROUP_ID} --tags Key=Name,Value=k8s
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol all --cidr 10.240.0.0/24
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol all --cidr 10.200.0.0/16
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol tcp --port 6443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol tcp --port 443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id ${SECURITY_GROUP_ID} --protocol icmp --port -1 --cidr 0.0.0.0/0

### Route Tables

Create a route tables that allows outbound communication across all protocols:
```
ROUTE_TABLE_ID=$(aws ec2 create-route-table --vpc-id ${VPC_ID} --output text --query 'RouteTable.RouteTableId')
aws ec2 create-route --route-table-id <your-rtb-id> --destination-cidr-block 0.0.0.0/0 --gateway-id <your-igw-id>
aws ec2 create-tags --resources ${ROUTE_TABLE_ID} --tags Key=Name,Value=kubernetes
```
To make this working we need now to get our subnets and attach them to the route table. Also we will allow all outbound traffic.
```
aws ec2 associate-route-table --route-table-id ${ROUTE_TABLE_ID} --subnet-id ${SUBNET_ID}
aws ec2 create-route --route-table-id ${ROUTE_TABLE_ID} --destination-cidr-block 0.0.0.0/0 --gateway-id ${INTERNET_GATEWAY_ID}

```

### Load Balancing
To give a reliable endpoint for the K8s API we will run an eleastic load balancer (ELB).
```
 LOAD_BALANCER_ARN=$(aws elbv2 create-load-balancer \
    --name k8s \
    --subnets ${SUBNET_ID} \
    --scheme internet-facing \
    --type network \
    --output text --query 'LoadBalancers[].LoadBalancerArn')

  TARGET_GROUP_ARN=$(aws elbv2 create-target-group \
    --name k8s \
    --protocol TCP \
    --port 6443 \
    --vpc-id ${VPC_ID} \
    --target-type ip \
    --output text --query 'TargetGroups[].TargetGroupArn')

  aws elbv2 register-targets --target-group-arn ${TARGET_GROUP_ARN} --targets Id=10.240.0.1{0,1,2}
  aws elbv2 create-listener \
    --load-balancer-arn ${LOAD_BALANCER_ARN} \
    --protocol TCP \
    --port 443 \
    --default-actions Type=forward,TargetGroupArn=${TARGET_GROUP_ARN} \
    --output text --query 'Listeners[].ListenerArn'
```
## Compute Instances

The compute instances in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 18.04, which has good support for the [containerd container runtime](https://github.com/containerd/containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

### Key Pair
To be able to ssh later on to the EC2 machines we need a key pair and save this localy.

```
aws ec2 create-key-pair --key-name K8sKeys --query 'KeyMaterial' --output text > K8sKeys.pem
chmod 400 K8sKeys.pem
```
### Kubernetes Controllers

Create three compute instances which will host the Kubernetes control plane:
> The AMI ID depends on your region. You can find the right AMI at [ubuntu.com](https://cloud-images.ubuntu.com/locator/ec2/). We use here an AMI for eu-central-1 aka Frankfurt.

```
${IMAGE_ID}=ami-0d3d3732ee4ccceff
```
We will use here t2.micros, which is in the free tier inclueded. However there can be still some costs, so don't forget to tidy up later.

```
for i in 0 1 2; do
  instance_id=$(aws ec2 run-instances \
    --associate-public-ip-address \
    --image-id ${IMAGE_ID} \
    --count 1 \
    --key-name K8sKeys \
    --security-group-ids ${SECURITY_GROUP_ID} \
    --instance-type t2.micro \
    --private-ip-address 10.240.0.1${i} \
    --user-data "name=controller-${i}" \
    --subnet-id ${SUBNET_ID} \
    --block-device-mappings='{"DeviceName": "/dev/sda1", "Ebs": { "VolumeSize": 20 }, "NoDevice": "" }' \
    --output text --query 'Instances[].InstanceId')
  aws ec2 modify-instance-attribute --instance-id ${instance_id} --no-source-dest-check
  aws ec2 create-tags --resources ${instance_id} --tags "Key=Name,Value=controller-${i}"
  echo "controller-${i} created "
done
```


### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr` instance metadata will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three compute instances which will host the Kubernetes worker nodes:

```
for i in 0 1 2; do
  instance_id=$(aws ec2 run-instances \
    --associate-public-ip-address \
    --image-id ${IMAGE_ID} \
    --count 1 \
    --key-name k8s \
    --security-group-ids ${SECURITY_GROUP_ID} \
    --instance-type t2.micro \
    --private-ip-address 10.240.0.2${i} \
    --user-data "name=worker-${i}|pod-cidr=10.200.${i}.0/24" \
    --subnet-id ${SUBNET_ID} \
    --block-device-mappings='{"DeviceName": "/dev/sda1", "Ebs": { "VolumeSize": 20 }, "NoDevice": "" }' \
    --output text --query 'Instances[].InstanceId')
  aws ec2 modify-instance-attribute --instance-id ${instance_id} --no-source-dest-check
  aws ec2 create-tags --resources ${instance_id} --tags "Key=Name,Value=worker-${i}"
  echo "worker-${i} created"
done
```

### Verification

List the compute instances in your default compute zone:

```
aws ec2 describe-instances --filters "Name=tag:Name,Values=worker-0,worker-1,worker-2,controller-0,controller-1,controller-2"
```

> output

```

```


### Important Note
The here designed VPC and network is just for an easy start with Kubernetes and doesn't realize the [Well Architected Framework](https://aws.amazon.com/architecture/well-architected/).  

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
