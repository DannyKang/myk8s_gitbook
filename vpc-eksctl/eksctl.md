# eksctl 구성

## eksctl 소개

`eksctl`은 관리형 Kubernetes 서비스 인 EKS에서 클러스터를 생성하기위한 간단한 CLI 도구입니다. Go로 작성되었으며 CloudFormation을 사용하며 [Weaveworks](https://www.weave.works/) 가 작성했으며 단 하나의 명령으로 몇 분 안에 기본 클러스터를 만듭니다.

## eksctl을 통한 EKS 구성

### 1.VPC/Subnet 정보 확인

앞서 [Cloudformation 구성](cloudformation.md#3-stack)에서 생성한 VPC id, Subnet id를 확인합니다.

아래는 예제입니다.

```text
Public subnet 01 - subnet-0e85e68a85894cf50
Public subnet 02 - subnet-0b8a862c807869d6b
Public subnet 03 - subnet-019d103bacc5dcb21
Private subnet 01 - subnet-0eb13d8b433c106a2
Private subnet 02 - subnet-0b188593b2d1755ce
Private subnet 03 - subnet-0a7fb1ebc6b148035
VPC ID - vpc-099a900fe8dde7319
```

### 2. eksctl 배포 yaml 다운로드

Cloud9에서 eksctl 배포용 yaml파일을 다운로드 받습니다.

```text
git clone https://github.com/whchoi98/myeks.git
```

Cloud9 IDE 편집기에서 아래와 같이 수정합니다. 수정내용은 현재 생성된 VPC, Subnet ID , key 위치 입니다.

![](../.gitbook/assets/image%20%282%29.png)

수정할 블록의 예시입니다.

```text
vpc:
  id: vpc-099a900fe8dde7319

public
ap-northeast-2a: { id: subnet-0e85e68a85894cf50}
ap-northeast-2b: { id: subnet-0b8a862c807869d6b}
ap-northeast-2c: { id: subnet-019d103bacc5dcb21}

private
ap-northeast-2a: { id: subnet-0eb13d8b433c106a2}
ap-northeast-2b: { id: subnet-0b188593b2d1755ce}
ap-northeast-2c: { id: subnet-0a7fb1ebc6b148035}

publicKeyPath: "/home/ec2-user/environment/eksworkshop.pub"
```

### 3. cluster 생성

eksctl을 통해 EKS Cluster를 생성합니다.

```text
eksctl create cluster --config-file=/home/ec2-user/environment/myeks/whchoi-cluster.yaml 
```

{% hint style="info" %}
Cluster를 생성하기 위해 20~30분 정도 시간이 소요됩니다.
{% endhint %}

출력 결과 예시

```text
whchoi98:~/environment $ eksctl create cluster --config-file=/home/ec2-user/environment/myeks/whchoi-cluster.yaml
[ℹ]  eksctl version 0.23.0
[ℹ]  using region ap-northeast-2
[✔]  using existing VPC (vpc-086186d07739fc568) and subnets (private:[subnet-051b3655d99b2cf0b subnet-0e3c5d12ade472f91 subnet-0dd1d98a956bf8227] public:[subnet-0d4864467efbf04a4 subnet-0662f5c059aac8576 subnet-0253297add231a70d])
[!]  custom VPC/subnets will be used; if resulting cluster doesn't function as expected, make sure to review the configuration of VPC/subnets
[ℹ]  nodegroup "ng1-public" will use "ami-0801b8b3cce955050" [AmazonLinux2/1.16]
[ℹ]  using SSH public key "/home/ec2-user/environment/eksworkshop.pub" as "eksctl-eksworkshop-nodegroup-ng1-public-02:ff:18:ce:c0:1f:0f:60:77:af:2a:a4:5a:60:d4:88" 
[ℹ]  nodegroup "ng2-private" will use "ami-0801b8b3cce955050" [AmazonLinux2/1.16]
[ℹ]  using SSH public key "/home/ec2-user/environment/eksworkshop.pub" as "eksctl-eksworkshop-nodegroup-ng2-private-02:ff:18:ce:c0:1f:0f:60:77:af:2a:a4:5a:60:d4:88" 
[ℹ]  using Kubernetes version 1.16
[ℹ]  creating EKS cluster "eksworkshop" in "ap-northeast-2" region with un-managed nodes
[ℹ]  2 nodegroups (ng1-public, ng2-private) were included (based on the include/exclude rules)
[ℹ]  will create a CloudFormation stack for cluster itself and 2 nodegroup stack(s)
[ℹ]  will create a CloudFormation stack for cluster itself and 0 managed nodegroup stack(s)
[ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=ap-northeast-2 --cluster=eksworkshop'
[ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "eksworkshop" in "ap-northeast-2"
[ℹ]  2 sequential tasks: { create cluster control plane "eksworkshop", 2 sequential sub-tasks: { update CloudWatch logging configuration, 2 parallel sub-tasks: { create nodegroup "ng1-public", create nodegroup "ng2-private" } } }
[ℹ]  building cluster stack "eksctl-eksworkshop-cluster"
[ℹ]  deploying stack "eksctl-eksworkshop-cluster"
[✔]  configured CloudWatch logging for cluster "eksworkshop" in "ap-northeast-2" (enabled types: api, audit, authenticator, controllerManager, scheduler & no types disabled)
[ℹ]  building nodegroup stack "eksctl-eksworkshop-nodegroup-ng2-private"
[ℹ]  building nodegroup stack "eksctl-eksworkshop-nodegroup-ng1-public"
[ℹ]  deploying stack "eksctl-eksworkshop-nodegroup-ng1-public"
[ℹ]  deploying stack "eksctl-eksworkshop-nodegroup-ng2-private"
[ℹ]  waiting for the control plane availability...
[✔]  saved kubeconfig as "/home/ec2-user/.kube/config"
[ℹ]  no tasks
[✔]  all EKS cluster resources for "eksworkshop" have been created
[ℹ]  adding identity "arn:aws:iam::909121566064:role/eksctl-eksworkshop-nodegroup-ng1-NodeInstanceRole-1970I5BJYVPFS" to auth ConfigMap
[ℹ]  nodegroup "ng1-public" has 0 node(s)
[ℹ]  waiting for at least 3 node(s) to become ready in "ng1-public"
[ℹ]  nodegroup "ng1-public" has 3 node(s)
[ℹ]  node "ip-10-11-31-153.ap-northeast-2.compute.internal" is ready
[ℹ]  node "ip-10-11-53-186.ap-northeast-2.compute.internal" is ready
[ℹ]  node "ip-10-11-69-28.ap-northeast-2.compute.internal" is ready
[ℹ]  adding identity "arn:aws:iam::909121566064:role/eksctl-eksworkshop-nodegroup-ng2-NodeInstanceRole-EHKBO7F93TJF" to auth ConfigMap
[ℹ]  nodegroup "ng2-private" has 0 node(s)
[ℹ]  waiting for at least 3 node(s) to become ready in "ng2-private"
[ℹ]  nodegroup "ng2-private" has 3 node(s)
[ℹ]  node "ip-10-11-114-132.ap-northeast-2.compute.internal" is ready
[ℹ]  node "ip-10-11-146-170.ap-northeast-2.compute.internal" is ready
[ℹ]  node "ip-10-11-189-67.ap-northeast-2.compute.internal" is ready
[ℹ]  kubectl command should work with "/home/ec2-user/.kube/config", try 'kubectl get nodes'
[✔]  EKS cluster "eksworkshop" in "ap-northeast-2" region is ready
```

### 4. Cluster 생성 확인

정상적으로 Cluster가 생성되었는지 확인합니다.

```text
kubectl get nodes
```

출력 결과 예시

```text
whchoi98:~/environment $ kubectl get nodes
NAME                                               STATUS   ROLES    AGE    VERSION
ip-10-11-114-132.ap-northeast-2.compute.internal   Ready    <none>   2d1h   v1.16.12-eks-904af05
ip-10-11-146-170.ap-northeast-2.compute.internal   Ready    <none>   2d1h   v1.16.12-eks-904af05
ip-10-11-189-67.ap-northeast-2.compute.internal    Ready    <none>   2d1h   v1.16.12-eks-904af05
ip-10-11-31-153.ap-northeast-2.compute.internal    Ready    <none>   2d1h   v1.16.12-eks-904af05
ip-10-11-53-186.ap-northeast-2.compute.internal    Ready    <none>   2d1h   v1.16.12-eks-904af05
ip-10-11-69-28.ap-northeast-2.compute.internal     Ready    <none>   2d1h   v1.16.12-eks-904af05
```

eksworkshop에서 사용될 Stack Name, Work Node Role Name을 환경변수에 저장해 둡니다.

```text
STACK_NAME=$(eksctl get nodegroup --cluster eksworkshop -o json | jq -r '.[].StackName')
ROLE_NAME=$(aws cloudformation describe-stack-resources --stack-name $STACK_NAME | jq -r '.StackResources[] | select(.ResourceType=="AWS::IAM::Role") | .PhysicalResourceId')
echo "export ROLE_NAME=${ROLE_NAME}" | tee -a ~/.bash_profile
```

생성된 VPC와 Subnet, Internet Gateway, NAT Gateway, Route Table등을 확인해 봅니다.

생성된 EC2 Worker Node들도 확인해 봅니다.

EKS와 eksctl을 통해 생생된 Cloudformation도 확인해 봅니다.
