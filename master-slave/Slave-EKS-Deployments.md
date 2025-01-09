# Jenkins Slave Setup Script for AWS EKS Deployments

## Installation Script
```bash
#!/bin/bash

# Exit on error
set -e

# Variables
KUBECTL_VERSION="1.27.1"
HELM_VERSION="3.12.0"
AWS_CLI_VERSION="2.11.0"
JENKINS_USER="jenkins"
WORKSPACE_DIR="/opt/jenkins/workspace"

echo "Starting Jenkins slave setup for EKS deployments..."

# Update system
echo "Updating system packages..."
sudo apt-get update
sudo apt-get upgrade -y

# Install essential packages
echo "Installing essential packages..."
sudo apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common \
    python3-pip \
    jq \
    git \
    unzip \
    wget \
    gnupg \
    lsb-release

# Install Java
echo "Installing Java..."
sudo apt-get install -y openjdk-11-jdk

# Create Jenkins user and directory structure
echo "Setting up Jenkins user and directories..."
sudo useradd -m -s /bin/bash ${JENKINS_USER} || true
sudo mkdir -p ${WORKSPACE_DIR}
sudo chown -R ${JENKINS_USER}:${JENKINS_USER} ${WORKSPACE_DIR}

# Install AWS CLI v2
echo "Installing AWS CLI..."
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64-${AWS_CLI_VERSION}.zip" -o "awscliv2.zip"
unzip -o awscliv2.zip
sudo ./aws/install --update
rm -rf aws awscliv2.zip

# Install kubectl
echo "Installing kubectl..."
curl -LO "https://dl.k8s.io/release/v${KUBECTL_VERSION}/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl

# Install Helm
echo "Installing Helm..."
wget https://get.helm.sh/helm-v${HELM_VERSION}-linux-amd64.tar.gz
tar xvf helm-v${HELM_VERSION}-linux-amd64.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/
rm -rf linux-amd64 helm-v${HELM_VERSION}-linux-amd64.tar.gz

# Install eksctl
echo "Installing eksctl..."
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
sudo chmod +x /usr/local/bin/eksctl

# Install Docker
echo "Installing Docker..."
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo \
  "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io

# Configure Docker permissions
sudo usermod -aG docker ${JENKINS_USER}

# Install AWS IAM Authenticator
echo "Installing AWS IAM Authenticator..."
curl -o aws-iam-authenticator https://s3.us-west-2.amazonaws.com/amazon-eks/1.21.2/2021-07-05/bin/linux/amd64/aws-iam-authenticator
sudo chmod +x aws-iam-authenticator
sudo mv aws-iam-authenticator /usr/local/bin

# Set up SSH for Jenkins
echo "Setting up SSH for Jenkins..."
sudo -u ${JENKINS_USER} mkdir -p /home/${JENKINS_USER}/.ssh
sudo -u ${JENKINS_USER} ssh-keygen -t rsa -f /home/${JENKINS_USER}/.ssh/id_rsa -N ""
sudo chown -R ${JENKINS_USER}:${JENKINS_USER} /home/${JENKINS_USER}/.ssh

# Create kubeconfig directory
echo "Setting up kubeconfig directory..."
sudo -u ${JENKINS_USER} mkdir -p /home/${JENKINS_USER}/.kube
sudo chown -R ${JENKINS_USER}:${JENKINS_USER} /home/${JENKINS_USER}/.kube

# Create AWS credentials directory
echo "Setting up AWS credentials directory..."
sudo -u ${JENKINS_USER} mkdir -p /home/${JENKINS_USER}/.aws
sudo chown -R ${JENKINS_USER}:${JENKINS_USER} /home/${JENKINS_USER}/.aws

# Add environment variables
echo "Setting up environment variables..."
cat << EOF | sudo tee /etc/profile.d/jenkins-env.sh
export PATH=\$PATH:/usr/local/bin
export KUBECONFIG=/home/${JENKINS_USER}/.kube/config
EOF

# Create AWS credentials template
echo "Creating AWS credentials template..."
cat << EOF | sudo -u ${JENKINS_USER} tee /home/${JENKINS_USER}/.aws/credentials.template
[default]
aws_access_key_id = YOUR_ACCESS_KEY
aws_secret_access_key = YOUR_SECRET_KEY
region = YOUR_REGION
EOF

# Create helper scripts
echo "Creating helper scripts..."

# EKS cluster connection script
cat << 'EOF' | sudo tee /usr/local/bin/connect-eks
#!/bin/bash
CLUSTER_NAME=$1
REGION=$2

if [ -z "$CLUSTER_NAME" ] || [ -z "$REGION" ]; then
    echo "Usage: connect-eks <cluster-name> <region>"
    exit 1
fi

aws eks update-kubeconfig --name $CLUSTER_NAME --region $REGION
EOF
sudo chmod +x /usr/local/bin/connect-eks

# Cleanup script
echo "Creating cleanup script..."
cat << 'EOF' | sudo tee /usr/local/bin/cleanup-workspace
#!/bin/bash
WORKSPACE_DIR="/opt/jenkins/workspace"
DAYS_OLD=7

find $WORKSPACE_DIR -type d -mtime +$DAYS_OLD -exec rm -rf {} +
docker system prune -af --volumes
EOF
sudo chmod +x /usr/local/bin/cleanup-workspace

# Set up cron job for cleanup
echo "Setting up cleanup cron job..."
(crontab -l 2>/dev/null; echo "0 0 * * * /usr/local/bin/cleanup-workspace") | crontab -

# Create health check script
echo "Creating health check script..."
cat << 'EOF' | sudo tee /usr/local/bin/check-eks-tools
#!/bin/bash
echo "Checking EKS tools installation..."

# Check required tools
tools=("kubectl" "helm" "aws" "docker" "eksctl" "aws-iam-authenticator")
for tool in "${tools[@]}"; do
    if command -v $tool &> /dev/null; then
        echo "✓ $tool installed: $($tool version 2>&1 | head -n 1)"
    else
        echo "✗ $tool not found"
    fi
done

# Check Docker service
if systemctl is-active --quiet docker; then
    echo "✓ Docker service is running"
else
    echo "✗ Docker service is not running"
fi

# Check AWS credentials
if [ -f ~/.aws/credentials ]; then
    echo "✓ AWS credentials file exists"
else
    echo "✗ AWS credentials file not found"
fi

# Check kubeconfig
if [ -f ~/.kube/config ]; then
    echo "✓ Kubeconfig file exists"
else
    echo "✗ Kubeconfig file not found"
fi
EOF
sudo chmod +x /usr/local/bin/check-eks-tools

echo "Installation complete! Please configure:"
echo "1. AWS credentials in /home/${JENKINS_USER}/.aws/credentials"
echo "2. Kubeconfig for your EKS cluster"
echo "3. Run 'check-eks-tools' to verify installation"
```

## Post-Installation Configuration

### 1. AWS Credentials Setup
```bash
# Copy template and fill in credentials
sudo -u jenkins cp /home/jenkins/.aws/credentials.template /home/jenkins/.aws/credentials
sudo -u jenkins vim /home/jenkins/.aws/credentials
```

### 2. Jenkins Agent Configuration
```groovy
// In Jenkins master, configure the node with these labels
node {
    name 'eks-slave'
    description 'EKS deployment slave'
    numExecutors 2
    labelString 'eks docker kubernetes aws'
    remoteFS '/opt/jenkins/workspace'
}
```

### 3. Pipeline Example
```groovy
pipeline {
    agent {
        label 'eks'
    }
    
    environment {
        CLUSTER_NAME = 'your-eks-cluster'
        REGION = 'us-west-2'
    }
    
    stages {
        stage('Configure kubectl') {
            steps {
                sh """
                    aws eks update-kubeconfig \
                        --name ${CLUSTER_NAME} \
                        --region ${REGION}
                """
            }
        }
        
        stage('Deploy to EKS') {
            steps {
                sh """
                    helm upgrade --install my-app ./helm-charts \
                        --namespace production \
                        --set image.tag=${BUILD_NUMBER}
                """
            }
        }
    }
}
```

### 4. Security Considerations
- Use IAM roles with minimum required permissions
- Rotate AWS credentials regularly
- Keep tools and packages updated
- Monitor slave resources and cleanup regularly
- Use private subnets for slave instances
- Implement proper security groups

### 5. Maintenance Tasks
1. Regular updates:
```bash
# Update system and tools
sudo apt-get update && sudo apt-get upgrade -y
sudo snap refresh kubectl
helm repo update
```

2. Cleanup:
```bash
# Run cleanup script
cleanup-workspace
```

3. Health check:
```bash
# Verify tools installation
check-eks-tools
```

These scripts and configurations ensure:
- Proper setup for EKS deployments
- Required tools installation
- Security best practices
- Regular maintenance
- Resource optimization
