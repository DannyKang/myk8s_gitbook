---
description: 'Update: 2022-05-08'
---

# 스케쥴링-Karpenter

## Karpenter 소개&#x20;

![](<../.gitbook/assets/image (229) (1).png>)

Karpenter는 Kubernetes 클러스터의 애플리케이션을 처리하는 데 적합한 컴퓨팅 리소스만 자동으로 시작하고 빠르고 간단한 컴퓨팅 프로비저닝으로 클라우드를 최대한 활용할 수 있도록 설계 되었습니다.&#x20;

Karpenter는 AWS로 구축된 유연한 오픈 소스의 고성능 Kubernetes 클러스터 오토스케일러입니다. 애플리케이션 로드의 변화에 대응하여 적절한 크기의 컴퓨팅 리소스를 신속하게 실행함으로써 애플리케이션 가용성과 클러스터 효율성을 개선할 수 있습니다. 또한 Karpenter는 애플리케이션의 요구 사항을 충족하는 컴퓨팅 리소스를 적시에 제공하며, 앞으로 클러스터의 컴퓨팅 리소스 공간을 자동으로 최적화하여 비용을 절감하고 성능을 개선 할수 있습니다.&#x20;

Karpenter 이전에는 Kubernetes 사용자가 [Amazon EC2 Auto Scaling 그룹](https://docs.aws.amazon.com/autoscaling/ec2/userguide/AutoScalingGroup.html)과 [Kubernetes Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler)를 사용하는 애플리케이션을 지원하기 위해 클러스터의 컴퓨팅 파워를 동적으로 조정해야 했습니다. EKS를 사용하는 많은 고객들이 Kubernetes Cluster Autoscaler를 사용하여 클러스터 Auto Scaling을 구성하기가 어렵고 구성할 수 있는 범위가 제한적인 것에 대해 개선을 요구했습니다&#x20;

Karpenter가 클러스터에 설치되면 Karpenter는 예약되지 않은 포드의 전체 리소스 요청을 관찰하고 새 노드를 시작하고 종료하는 결정을 내림으로써 예약 대기 시간과 인프라 비용을 줄입니다. 이를 위해 Karpenter는 Kubernetes 클러스터 내의 이벤트를 관찰한 다음 Amazon EC2와 같은 기본 클라우드 공급자의 컴퓨팅 서비스로 명령을 전송합니다.

![](<../.gitbook/assets/image (236) (1) (1) (1) (1).png>)

Karpenter는 [Apache License 2.0](https://github.com/awslabs/karpenter/blob/main/LICENSE)을 통해 라이선스가 부여되는 오픈 소스 프로젝트입니다. 모든 주요 클라우드 공급업체 및 온프레미스 환경을 포함하여, 모든 환경에서 실행되는 모든 Kubernetes 클러스터와 함께 작동하도록 설계되었습니다.&#x20;



## Karpenter 설치

0\. EKS Cluster 설치

ap-northeast-1 region에 새로운 Cluster를 설치합니다.

* ap-northeast-1 region에서 새로운 Cloud9 을 설치 합니다. ([2.EKS 환경 구성 참조](../eks/cloud9-ide.md#1.-cloud9-ide))

설치  Script는 아래와 같이 진행합니다.

```
### AWS Cli 2.0 Install

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

source ~/.bashrc
aws --version

which aws_completer
export PATH=/usr/local/bin:$PATH
source ~/.bash_profile
complete -C '/usr/local/bin/aws_completer' aws

### Kubectl 1.21.5 Install
cd ~
curl -LO https://storage.googleapis.com/kubernetes-release/release/v1.21.5/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
source <(kubectl completion bash)
echo "source <(kubectl completion bash)" >> ~/.bashrc

### util setup

sudo yum -y install jq gettext bash-completion moreutils
for command in kubectl jq envsubst aws
  do
    which $command &>/dev/null && echo "$command in path" || echo "$command NOT FOUND"
  done


```



Cloud9 에 새로운 Role을 부여합니다. ([2.EKS 환경 구성 참조](../eks/cloud9-ide.md#1.-cloud9-ide))

```
rm -vf ${HOME}/.aws/credentials
aws sts get-caller-identity --region ap-northeast-1 --query Arn | grep eksworkshop-admin -q && echo "IAM role valid" || echo "IAM role NOT valid"
export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')
echo "export ACCOUNT_ID=${ACCOUNT_ID}" | tee -a ~/.bash_profile
echo "export AWS_REGION=${AWS_REGION}" | tee -a ~/.bash_profile
aws configure set default.region ${AWS_REGION}
aws configure --profile default list

```



ssh key를 설정하고, key를 전송합니다.

```
### SSH Key
cd ~/environment/
ssh-keygen

# key naem 은 eksworkshop으로 선
cd ~/environment/
mv ./eksworkshop ./eksworkshop.pem
chmod 400 ./eksworkshop.pem

cd ~/environment/
aws ec2 import-key-pair --key-name "eksworkshop" --public-key-material fileb://./eksworkshop.pub

```



KMS 를 설정합니다.

```
aws kms create-alias --alias-name alias/eksworkshop --target-key-id $(aws kms create-key --query KeyMetadata.Arn --output text)
export MASTER_ARN=$(aws kms describe-key --key-id alias/eksworkshop --query KeyMetadata.Arn --output text)
cd ~/environment
echo $MASTER_ARN
echo $MASTER_ARN > master_arn.txt
cat master_arn.txt
echo "export MASTER_ARN=${MASTER_ARN}" | tee -a ~/.bash_profile
```



```
### git clone
cd ~/environment
git clone https://github.com/whchoi98/myeks

cd ./myeks/

cd ./myeks/

aws cloudformation deploy \
  --stack-name "eksworkshop" \
  --template-file "karpenter_vpc.yml" \
  --capabilities CAPABILITY_NAMED_IAM 

```



```
# eksctl 설정 
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
# eksctl 자동완성 - bash
. <(eksctl completion bash)
eksctl version

```



```
### make env for the karpenter test      

export ekscluster_name=eksworkshop
export ACCOUNT_ID=$(aws sts get-caller-identity --region ap-northeast-1 --output text --query Account)
export k_public_mgmd_node="k-managed-frontend-workloads"
export k_private_mgmd_node="k-managed-backend-workloads"
export KARPENTER_VERSION="v0.13.2"
export eks_version="1.21"
export publicKeyPath="/home/ec2-user/environment/eksworkshop.pub"
echo ${ekscluster_name}
echo ${ACCOUNT_ID}
echo ${k_public_mgmd_node}
echo ${k_private_mgmd_node}
echo ${CLUSTER_ENDPOINT}
echo ${KARPENTER_VERSION}
echo "export ekscluster_name=${ekscluster_name}" | tee -a ~/.bash_profile
echo "export k_public_mgmd_node=${k_public_mgmd_node}" | tee -a ~/.bash_profile
echo "export k_private_mgmd_node=${k_private_mgmd_node}" | tee -a ~/.bash_profile
echo "export k_private_mgmd_node=${CLUSTER_ENDPOINT}" | tee -a ~/.bash_profile
echo "export eks_version=${eks_version}" | tee -a ~/.bash_profile
echo "export KARPENTER_VERSION=${KARPENTER_VERSION}" | tee -a ~/.bash_profile
source ~/.bash_profile

```



```
### VPC 정보 
cd ~/environment/
#VPC ID export
export vpc_ID=$(aws ec2 describe-vpcs --filters Name=tag:Name,Values=eksworkshop | jq -r '.Vpcs[].VpcId')
echo $vpc_ID

#Subnet ID, CIDR, Subnet Name export
aws ec2 describe-subnets --filter Name=vpc-id,Values=$vpc_ID | jq -r '.Subnets[]|.SubnetId+" "+.CidrBlock+" "+(.Tags[]|select(.Key=="Name").Value)'
echo $vpc_ID > vpc_subnet.txt
aws ec2 describe-subnets --filter Name=vpc-id,Values=$vpc_ID | jq -r '.Subnets[]|.SubnetId+" "+.CidrBlock+" "+(.Tags[]|select(.Key=="Name").Value)' >> vpc_subnet.txt
cat vpc_subnet.txt

# VPC, Subnet ID 환경변수 저장 
export PublicSubnet01=$(aws ec2 describe-subnets --filter Name=vpc-id,Values=$vpc_ID | jq -r '.Subnets[]|.SubnetId+" "+.CidrBlock+" "+(.Tags[]|select(.Key=="Name").Value)' | awk '/eksworkshop-PublicSubnet01/{print $1}')
export PublicSubnet02=$(aws ec2 describe-subnets --filter Name=vpc-id,Values=$vpc_ID | jq -r '.Subnets[]|.SubnetId+" "+.CidrBlock+" "+(.Tags[]|select(.Key=="Name").Value)' | awk '/eksworkshop-PublicSubnet02/{print $1}')
export PublicSubnet03=$(aws ec2 describe-subnets --filter Name=vpc-id,Values=$vpc_ID | jq -r '.Subnets[]|.SubnetId+" "+.CidrBlock+" "+(.Tags[]|select(.Key=="Name").Value)' | awk '/eksworkshop-PublicSubnet03/{print $1}')
export PrivateSubnet01=$(aws ec2 describe-subnets --filter Name=vpc-id,Values=$vpc_ID | jq -r '.Subnets[]|.SubnetId+" "+.CidrBlock+" "+(.Tags[]|select(.Key=="Name").Value)' | awk '/eksworkshop-PrivateSubnet01/{print $1}')
export PrivateSubnet02=$(aws ec2 describe-subnets --filter Name=vpc-id,Values=$vpc_ID | jq -r '.Subnets[]|.SubnetId+" "+.CidrBlock+" "+(.Tags[]|select(.Key=="Name").Value)' | awk '/eksworkshop-PrivateSubnet02/{print $1}')
export PrivateSubnet03=$(aws ec2 describe-subnets --filter Name=vpc-id,Values=$vpc_ID | jq -r '.Subnets[]|.SubnetId+" "+.CidrBlock+" "+(.Tags[]|select(.Key=="Name").Value)' | awk '/eksworkshop-PrivateSubnet03/{print $1}')
echo "export vpc_ID=${vpc_ID}" | tee -a ~/.bash_profile
echo "export PublicSubnet01=${PublicSubnet01}" | tee -a ~/.bash_profile
echo "export PublicSubnet02=${PublicSubnet02}" | tee -a ~/.bash_profile
echo "export PublicSubnet03=${PublicSubnet03}" | tee -a ~/.bash_profile
echo "export PrivateSubnet01=${PrivateSubnet01}" | tee -a ~/.bash_profile
echo "export PrivateSubnet02=${PrivateSubnet02}" | tee -a ~/.bash_profile
echo "export PrivateSubnet03=${PrivateSubnet03}" | tee -a ~/.bash_profile
echo "export publicKeyPath=${publicKeyPath}" | tee -a ~/.bash_profile
source ~/.bash_profile

```



```
### create nodegroup yaml file for the karpenter      

cat << EOF > ~/environment/myeks/karpenter-nodegroup.yaml
---
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: ${ekscluster_name}
  region: ${AWS_REGION}
  version: "${eks_version}"  
  tags:
    karpenter.sh/discovery: ${ekscluster_name}

iam:
  withOIDC: true

vpc: 
  id: ${vpc_ID}
  subnets:
    public:
      PublicSubnet01:
        az: ${AWS_REGION}a
        id: ${PublicSubnet01}
      PublicSubnet02:
        az: ${AWS_REGION}c
        id: ${PublicSubnet02}
      PublicSubnet03:
        az: ${AWS_REGION}d
        id: ${PublicSubnet03}
    private:
      PrivateSubnet01:
        az: ${AWS_REGION}a
        id: ${PrivateSubnet01}
      PrivateSubnet02:
        az: ${AWS_REGION}c
        id: ${PrivateSubnet02}
      PrivateSubnet03:
        az: ${AWS_REGION}d
        id: ${PrivateSubnet03}
secretsEncryption:
  keyARN: ${MASTER_ARN}

managedNodeGroups:
  - name: k-managed-ng-public-01
    instanceType: ${instance_type}
    subnets:
      - ${PublicSubnet01}
      - ${PublicSubnet02}
      - ${PublicSubnet03}
    desiredCapacity: 3
    minSize: 3
    maxSize: 6
    volumeSize: 200
    volumeType: gp3 
    amiFamily: AmazonLinux2
    labels:
      nodegroup-type: "${k_public_mgmd_node}"
    ssh: 
        publicKeyPath: "${publicKeyPath}"
        allow: true
    iam:
      attachPolicyARNs:
      withAddonPolicies:
        autoScaler: true
        cloudWatch: true
        ebs: true
        fsx: true
        efs: true
        
  - name: k-managed-ng-private-01
    instanceType: ${instance_type}
    subnets:
      - ${PrivateSubnet01}
      - ${PrivateSubnet02}
      - ${PrivateSubnet03}
    desiredCapacity: 3
    privateNetworking: true
    minSize: 3
    maxSize: 9
    volumeSize: 200
    volumeType: gp3 
    amiFamily: AmazonLinux2
    labels:
      nodegroup-type: "${k_private_mgmd_node}"
    ssh: 
        publicKeyPath: "${publicKeyPath}"
        allow: true
    iam:
      attachPolicyARNs:
      withAddonPolicies:
        autoScaler: true
        cloudWatch: true
        ebs: true
        fsx: true
        efs: true

EOF

### create nodegroup for the karpenter      
eksctl create nodegroup --config-file=/home/ec2-user/environment/myeks/karpenter-nodegroup.yaml

```

### 1.환경 설정

Kubernetes Metric-server를 설치합니다. 앞서 설치하였으면 생략합니다.&#x20;

```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

```

metric server가 정상적으로 설치가 완료되면 아래와 같이 리소스 모니터링을 확인 할 수 있습니다. K9s에서도 Pod들의 CPU/Memory 사용량을 확인 할 수 있습니다.&#x20;

```
# api service condition 상태를 확인해 봅니다 
kubectl get apiservice v1beta1.metrics.k8s.io -o yaml
# pod의 CPU/Memory가 모니터링되는지 확인해 봅니다
kubectl top pod --all-namespaces
```

Helm을 통해서 아래 kube-ops-view를 Cloud9에 설치합니다. 노드 배포를 확인하기 위해 kube-ops-view를 service type=LoadBalancer를 설치합니다.&#x20;

```
## 앞서 Lab에서 생성한 managed Node의 Public Subnet에 위치한 노드에 kube-ops-view를 설치합니다
cd ~/environment
curl -L https://git.io/get_helm.sh | bash -s -- --version v3.8.2

kubectl create namespace kube-tools
helm install kube-ops-view \
stable/kube-ops-view \
--namespace kube-tools \
--set service.type=LoadBalancer \
--set nodeSelector.nodegroup-type=managed-frontend-workloads \
--version 1.2.4 \
--set rbac.create=True

## loadbalancer의 FQDN을 확인하고 웹브라우저에서 접속해 봅니다 
kubectl -n kube-tools get svc kube-ops-view | tail -n 1 | awk '{ print "Kube-ops-view URL = http://"$4 }'

```

URL을 접속하면 , 아래와 같이 노드와 배치된 PoD들을 확인해 볼 수 있습니다

![](<../.gitbook/assets/image (224) (1) (1) (1).png>)

Karpenter 시험 환경 구성을 위해 아래와 같이 환경변수를 구성합니다.&#x20;

```
# EKS CLUSTER_ENDPOINT 값에 대한 환경변수 설정 
export CLUSTER_ENDPOINT="$(aws eks describe-cluster --name ${ekscluster_name} --query "cluster.endpoint" --output text)"
# KARPENTER_VERSION 설정
echo "export CLUSTER_ENDPOINT=${CLUSTER_ENDPOINT}" | tee -a ~/.bash_profile
source ~/.bash_profile

```

### 2.Karpenter 환경구

Subnet과 Security Group에 새로운 Tag를 설정합니다

```
aws ec2 create-tags --resources "$PublicSubnet01" --tags Key="karpenter.sh/discovery",Value="${ekscluster_name}" 
aws ec2 create-tags --resources "$PublicSubnet02" --tags Key="karpenter.sh/discovery",Value="${ekscluster_name}" 
aws ec2 create-tags --resources "$PublicSubnet03" --tags Key="karpenter.sh/discovery",Value="${ekscluster_name}"
 
```

### 3. Karpenter 노드 권한 설정

kubernetes와 IAM간 인증을 위해 OIDC Provider를 생성합니다. 이미 앞서 Lab에서 생성하였다면 생략합니다.&#x20;

```
eksctl utils associate-iam-oidc-provider \
    --region ${AWS_REGION} \
    --cluster ${ekscluster_name} \
    --approve
    
```

Karpenter Node들을 위한 IAM Role을 생성합니다. karpenter node를 위한 IAM Role Template을 다운로드 합니다.&#x20;

```
mkdir /home/ec2-user/environment/karpenter
export KARPENTER_CF="/home/ec2-user/environment/karpenter/k-node-iam-role.yaml"
echo ${KARPENTER_CF}

curl -fsSL https://karpenter.sh/"${KARPENTER_VERSION}"/getting-started/getting-started-with-eksctl/cloudformation.yaml  > $KARPENTER_CF
sed -i 's/\${ClusterName}/eksworkshop/g' $KARPENTER_CF
#eksworkshop은 앞서 정의한 eks clustername 입니다. 다르게 설정한 경우 다른 값을 입력합니다 
```

AWS CLI를 통해서 IAM Role 구성을 위한 Cloudformation을 배포합니다.&#x20;

```
cd ~/environment/karpenter
aws cloudformation deploy \
  --stack-name "Karpenter-${ekscluster_name}" \
  --template-file "${KARPENTER_CF}" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides ClusterName="${ekscluster_name}"
  
```

Karpenter Node들을 위해 생성된 IAM Role을 eksctl을 통해 kubernetes 권한에 Mapping 합니다.&#x20;

```
eksctl create iamidentitymapping \
  --username system:node:{{EC2PrivateDNSName}} \
  --cluster "${ekscluster_name}" \
  --arn "arn:aws:iam::${ACCOUNT_ID}:role/KarpenterNodeRole-${ekscluster_name}" \
  --group system:bootstrappers \
  --group system:nodes
  
```

Kube-system Configmap/aws-auth에 정상적으로 Mapping 되었는지 확인합니다.&#x20;

```
kubectl describe -n kube-system configmap/aws-auth 

```

아래와 같이 추가되었습니다.&#x20;

```
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::123834880106:role/eksctl-eksworkshop-nodegroup-k-ma-NodeInstanceRole-8MF84X3ZTWLY
      username: system:node:{{EC2PrivateDNSName}}
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::123834880106:role/eksctl-eksworkshop-nodegroup-k-ma-NodeInstanceRole-MH0HMELFYYQW
      username: system:node:{{EC2PrivateDNSName}}
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::123834880106:role/KarpenterNodeRole-eksworkshop
      username: system:node:{{EC2PrivateDNSName}}

```

### 4. Service Account 생성

eksctl로 Kubernetes Service Account를 생성하고, 앞서 생성한 IAM Role을  Mapping 합니다.&#x20;

```
eksctl create iamserviceaccount \
  --cluster "${ekscluster_name}" --name karpenter --namespace karpenter \
  --role-name "${ekscluster_name}-karpenter" \
  --attach-policy-arn "arn:aws:iam::${ACCOUNT_ID}:policy/KarpenterControllerPolicy-${ekscluster_name}" \
  --role-only \
  --override-existing-serviceaccounts \
  --approve
  
# KARPENTER IAM ROLE ARN을 변수에 저장해 둡니다. 
export KARPENTER_IAM_ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/${ekscluster_name}-karpenter"
echo "export KARPENTER_IAM_ROLE_ARN=${KARPENTER_IAM_ROLE_ARN}" | tee -a ~/.bash_profile
source ~/.bash_profile

```

EC2 Spot Service를 처음 활성화 시킨 다면, 아래를 구성합니다.&#x20;

```
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com || true

```

### 5. Karpenter 설치

Helm을 사용하여 Karpenter를 클러스터에 배포합니다.&#x20;

Helm Chart 를 설치하기 전에 Repo를 Helm에 추가해야 하므로 다음 명령을 실행하여 Repo를 추가합니다.

```
helm repo add karpenter https://charts.karpenter.sh/
helm repo update

```

Cluster의 상세 정보 및 Karpenter Role ARN을 전달하는  Helm Chart를 설치합니다.&#x20;

```
echo ${KARPENTER_VERSION}
echo ${KARPENTER_IAM_ROLE_ARN}
echo ${ekscluster_name}
echo ${CLUSTER_ENDPOINT}
 
helm upgrade --install --namespace karpenter --create-namespace \
  karpenter karpenter/karpenter \
  --version ${KARPENTER_VERSION} \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=${KARPENTER_IAM_ROLE_ARN} \
  --set clusterName=${ekscluster_name} \
  --set clusterEndpoint=${CLUSTER_ENDPOINT} \
  --set aws.defaultInstanceProfile=KarpenterNodeInstanceProfile-${ekscluster_name} \
  --wait 
  
  
  #  --set nodeSelector.intent=public-control-apps \
  #  --set defaultProvisioner.create=false \
```

karpenter pod가 정상적으로 설치 되었는지 확인합니다.&#x20;

```
kubectl get pods --namespace karpenter
kubectl get deployment -n karpenter

```

### 6.Provisioner 구성

Karpenter 구성은 Provisioner CRD(Custom Resource Definition) 형식으로 제공됩니다. 단일 Karpenter Provisioner는 다양한 Pod를 구성할 수 있습니다. Karpenter는 Label 및 Affinity와 같은 Pod의 속성을 기반으로 Scheduling 및 프로비저닝 결정을 할 수 있습니다. Karpenter는 다양한 노드 그룹을 관리할 필요가 없습니다.

아래 명령을 사용하여 기본 프로비저닝 도구를 만들기 위한 yaml을 정의합니다.이 프로비저닝 도구는 securityGroupSelector 및 subnetSelector를 사용하여 노드를 시작하는 데 사용되는 리소스를 검색합니다. 위의 eksctl 명령에 karpenter.sh/discovery 태그를 적용했습니다.&#x20;

앞서 Karpenter Node를 구성할 때 lable 설정으로 "control-apps"로 앞서 Node를 구분지었습니다.&#x20;

```
cat << EOF > ~/environment/karpenter/karpenter-provisioner.yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot"]
  limits:
    resources:
      cpu: 1000
  provider:
    subnetSelector:
      karpenter.sh/discovery: eksworkshop
    securityGroupSelector:
      karpenter.sh/discovery: eksworkshop
  ttlSecondsAfterEmpty: 30
EOF

## 생성된 provisioner.yaml을 실행합니다. 
kubectl apply -f ~/environment/karpenter/karpenter-provisioner.yaml

```

* **`instanceProfile`**: Karpenter에서 시작한 인스턴스는 컨테이너를 실행하고 네트워킹을 구성하는 데 필요한 권한을 부여하는 InstanceProfile로 실행해야 합니다.&#x20;
* **`requirements`** : Provisioner CRD는 인스턴스 type 및 AZ와 같은 노드 속성을 선택할 수 있습니다. 예를 들어 topology.kubernetes.io/zone=ap-northeast-2a 레이블에 대한 응답으로 Karpenter는 해당 가용성 영역에 노드를 프로비저닝합니다. 이 예에서는 karpenter.sh/capacity-type을 설정하여 EC2 스팟 인스턴스를 사용합니다. 여기에서 사용할 수 있는 다른 속성을 확인할 수 있습니다. 이번 랩에서 몇 가지 더 작업할 것입니다.&#x20;
* **`limit`** : provisioner는 클러스터에 할당된 CPU 및 메모리 수의 제한을 정의할 수 있습니다. ttlSecondsAfterEmpty: 값은 빈 노드를 종료하도록 Karpenter를 구성합니다.&#x20;
* **`provider:tags`** : EC2 인스턴스가 생성될 때 가지게 되는 Tag를 정의할 수도 있습니다. 이것은 EC2 수준에서 Billing 및 거버넌스를 활성화하는 데 도움이 됩니다.
* **`ttlSecondsAfterEmpty`** : 값은 Karpenter가 노드에 자원이 배치가 없는 경우 종료하도록 구성합니다.값을 정의하지 않은 상태로 두면 이 동작을 비활성화할 수 있습니다. 이 경우 빠른 시연을 위해 30초 값으로 설정했습니다.

### 7. 자동 노드 프로비저닝

Karpenter는 이제 활성화되었으며 노드 프로비저닝을 시작할 준비가 되었습니다. Deployment를 사용하여 Pod를 만들고 Karpenter가 노드를 프로비저닝하는 것을 확인해 봅니다.자동 노드 프로비저닝 이 배포는 [pause image](https://www.ianlewis.org/en/almighty-pause-container)를 사용하고 replica가  없는 상태에서 시작합니다.

Deployment 하기 전에 웹브라우저에서 앞서 생성한 kube-ops-view를 열어두고 배포를 살펴 봅니다.&#x20;

```
kubectl -n kube-tools get svc kube-ops-view | tail -n 1 | awk '{ print "Kube-ops-view URL = http://"$4 }'

```

```
kubectl create namespace karpenter-inflate
cat << EOF > ~/environment/karpenter/karpenter-inflate.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate
  namespace: karpenter-inflate  
spec:
  replicas: 0
  selector:
    matchLabels:
      app: inflate
  template:
    metadata:
      labels:
        app: inflate
    spec:
      terminationGracePeriodSeconds: 0
      containers:
        - name: inflate
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.2
          resources:
            requests:
              cpu: 1
      nodeSelector:
        karpenter.sh/capacity-type: spot
EOF
## 생성된 karpenter-inflate.yaml을 실행합니다. 
kubectl apply -f ~/environment/karpenter/karpenter-inflate.yaml

```

replica를 늘려가면서 시험해 봅니다.

```
kubectl -n karpenter-inflate scale deployment inflate --replicas 5

```

Terminal 에서 로그를 확인해 봅니다.

```
kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter -c controller

```

### 10. 자원 삭제 (Option)



```
kubectl delete namespace karpenter-inflate
kubectl delete -f ~/environment/karpenter/karpenter-provisioner.yaml
helm uninstall karpenter --namespace karpenter
aws iam detach-role-policy --role-name="${ekscluster_name}-karpenter" --policy-arn="arn:aws:iam::${ACCOUNT_ID}:policy/KarpenterControllerPolicy-${ekscluster_name}"
aws iam delete-policy --policy-arn="arn:aws:iam::${ACCOUNT_ID}:policy/KarpenterControllerPolicy-${ekscluster_name}"
aws iam delete-role --role-name="${ekscluster_name}-karpenter"
aws cloudformation delete-stack --stack-name "Karpenter-${ekscluster_name}"
aws ec2 describe-launch-templates \
    | jq -r ".LaunchTemplates[].LaunchTemplateName" \
    | grep -i "Karpenter-${ekscluster_name}" \
    | xargs -I{} aws ec2 delete-launch-template --launch-template-name {}
aws cloudformation delete-stack --stack-name eksctl-eksworkshop-addon-iamserviceaccount-karpenter-karpenter
 eksctl delete nodegroup --config-file=/home/ec2-user/environment/myeks/karpenter-nodegroup.yaml --approve

```





