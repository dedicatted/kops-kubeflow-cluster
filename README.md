sudo yum update
#Kops
wget -O kops https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64
chmod +x ./kops
sudo mv ./kops /usr/local/bin/

#Kubectl
wget -O kubectl https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl

#At this step, you need to specify a different name for "CLUSTER_NAME="
export CLUSTER_NAME=kops-kubeflow-cluster
export NAME=${CLUSTER_NAME}.k8s.local
export KOPS_STATE_STORE=s3://${CLUSTER_NAME}

#Create S3 bucket
aws s3api create-bucket \
--bucket ${CLUSTER_NAME} \
--region us-east-1

#############
#Create ssh key if need (key admin)
cd ~/.ssh/
ssh-keygen
eval $(ssh-agent -s) && ssh-add ~/.ssh/key
#############

kops create cluster \
--state ${KOPS_STATE_STORE} \
--zones us-east-1a \
--master-count 1 \
--node-count 2 \
--node-size t2.micro \
--master-size t2.micro \
${NAME}

kops create secret --name ${NAME} sshpublickey admin -i ~/.ssh/id_rsa.pub
kops update cluster ${NAME} --yes
#Wait 10 minutes and use this command
kops validate cluster
#If you see an error after executing the command, wait a few minutes and try again.
#You can continue if you see something like that "Your cluster <your_cluster_name> is ready"

#ksonet
export KS_VER=0.12.0
export KS_PKG=ks_${KS_VER}_linux_amd64
export PATH=$PATH:${HOME}/bin/$KS_PKG
wget -O /tmp/${KS_PKG}.tar.gz https://github.com/ksonnet/ksonnet/releases/download/v${KS_VER}/${KS_PKG}.tar.gz \
--no-check-certificate
mkdir -p ${HOME}/bin
tar -xvf /tmp/$KS_PKG.tar.gz -C ${HOME}/bin

#Kubeflow
export KUBEFLOW_SRC=/home/ec2-user/kubeflow
export KUBEFLOW_TAG=v0.3.1
export KFAPP=${KUBEFLOW_SRC}/app
mkdir ${KUBEFLOW_SRC} && mkdir ${KFAPP}
cd ${KUBEFLOW_SRC}

curl https://raw.githubusercontent.com/kubeflow/kubeflow/${KUBEFLOW_TAG}/scripts/download.sh | bash

${KUBEFLOW_SRC}/scripts/kfctl.sh init ${KFAPP} --platform aws
cd ${KFAPP}
${KUBEFLOW_SRC}/scripts/kfctl.sh generate k8s
${KUBEFLOW_SRC}/scripts/kfctl.sh apply k8s
