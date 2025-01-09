# Jenkins Master-Slave Architecture Setup Guide

## 1. Master Node Configuration

### System Requirements for Master Node
```properties
# hardware-requirements.txt
Minimum Requirements:
- CPU: 4 cores
- RAM: 8GB
- Disk: 50GB SSD
- Network: 1 Gbps

Recommended for Mid-size Projects:
- CPU: 8 cores
- RAM: 16GB
- Disk: 100GB SSD
- Network: 1 Gbps
```

### Jenkins Master Configuration
```groovy
// jenkins.config
jenkins {
    systemMessage = 'Jenkins Master Node'
    numExecutors = 0  // Master should not run jobs
    mode = Node.Mode.EXCLUSIVE
    labelString = 'master'
    
    securityRealm {
        local {
            allowsSignup = false
            enableCaptcha = false
        }
    }
    
    authorizationStrategy {
        roleBased {
            assignments {
                role('admin') {
                    users('admin')
                }
                role('developer') {
                    users('dev1', 'dev2')
                }
            }
        }
    }
}
```

## 2. Slave Node Setup

### Linux Slave Setup Script
```bash
#!/bin/bash

# Install Java
sudo apt update
sudo apt install -y openjdk-11-jdk

# Create Jenkins User
sudo useradd -m -s /bin/bash jenkins
sudo mkdir -p /opt/jenkins
sudo chown -R jenkins:jenkins /opt/jenkins

# Set up SSH Keys
sudo -u jenkins ssh-keygen -t rsa -f /home/jenkins/.ssh/id_rsa -N ""

# Install Required Tools
sudo apt install -y git docker.io maven npm

# Configure Docker permissions
sudo usermod -aG docker jenkins

# Create workspace directory
sudo mkdir -p /opt/jenkins/workspace
sudo chown -R jenkins:jenkins /opt/jenkins/workspace
```

### Windows Slave Setup Script
```powershell
# Install Chocolatey
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))

# Install Required Software
choco install -y openjdk11
choco install -y git
choco install -y docker-desktop
choco install -y maven
```

## 3. Master-Slave Connection Configuration

### SSH Method Configuration
```groovy
// Node configuration in Jenkins
node {
    name 'linux-slave-01'
    description 'Linux build slave'
    numExecutors 4
    remoteFS '/opt/jenkins/workspace'
    labelString 'linux docker maven'
    mode Mode.EXCLUSIVE
    launcher new SSHLauncher(
        'slave-hostname', // Hostname
        22,              // Port
        'credentials-id',// Jenkins credentials ID
        '',             // JVM Options
        '',             // JavaPath
        '',             // Prefix Start Slave Command
        '',             // Suffix Start Slave Command
        180,            // Connection Timeout in Seconds
        10,            // Maximum Number of Retries
        5              // Seconds To Wait Between Retries
    )
    retentionStrategy new RetentionStrategy.Always()
}
```

### JNLP Method Configuration
```xml
<!-- agent.xml -->
<slave>
  <name>windows-slave-01</name>
  <description>Windows build slave</description>
  <executors>4</executors>
  <label>windows dotnet msbuild</label>
  <launcher class="hudson.slaves.JNLPLauncher">
    <workDirSettings>
      <disabled>false</disabled>
      <internalDir>remoting</internalDir>
      <failIfWorkDirIsMissing>false</failIfWorkDirIsMissing>
    </workDirSettings>
  </launcher>
</slave>
```

## 4. Resource Allocation Strategy

### For Mid-level Projects (50-100 developers)

1. **Recommended Slave Count:**
   - 3-5 Linux slaves for general builds
   - 1-2 Windows slaves (if needed)
   - 1-2 specialized slaves (for specific builds)

2. **Resource Distribution:**
```yaml
# resource-allocation.yaml
general_purpose_slaves:
  count: 3
  specs_per_slave:
    cpu: 4 cores
    ram: 8GB
    disk: 100GB
    executors: 4

specialized_slaves:
  count: 2
  specs_per_slave:
    cpu: 8 cores
    ram: 16GB
    disk: 200GB
    executors: 2

windows_slaves:
  count: 1
  specs_per_slave:
    cpu: 4 cores
    ram: 16GB
    disk: 150GB
    executors: 2
```

## 5. Best Practices

### 1. Master Node
- Keep master node free from builds
- Regular backup of master configuration
- Monitoring of master resources
- Security hardening

### 2. Slave Management
```groovy
// Label and capacity-based distribution
pipeline {
    agent {
        label 'linux && docker'
    }
    options {
        timeout(time: 1, unit: 'HOURS')
    }
}
```

### 3. Resource Optimization
```groovy
// Executor management
properties([
    disableConcurrentBuilds(),
    durabilityHint('PERFORMANCE_OPTIMIZED')
])
```

### 4. Load Balancing
```groovy
// Load balancing configuration
jenkins.model.Jenkins.instance.nodes.each { node ->
    node.mode = Node.Mode.EXCLUSIVE
    node.numExecutors = 4
}
```

## 6. Scaling Guidelines

### When to Add More Slaves:
1. Build queue time > 10 minutes
2. CPU usage consistently > 80%
3. Memory usage consistently > 80%
4. Build times increasing

### Scaling Formula:
```python
# Rough calculation for required slaves
required_slaves = (peak_concurrent_builds * avg_build_time) / (executor_per_slave * build_window)
```

## 7. Common Issues and Solutions

### Network Issues
```bash
# Check network connectivity
ssh -v jenkins@slave-host

# Test port connectivity
nc -zv slave-host 22
```

### Permission Issues
```bash
# Fix workspace permissions
sudo chown -R jenkins:jenkins /opt/jenkins/workspace
sudo chmod -R 755 /opt/jenkins/workspace
```

### Memory Issues
```groovy
// Add memory parameters to slave launch
env.JENKINS_SLAVE_ARGS = """
    -Xmx2048m
    -XX:MaxPermSize=512m
    -XX:+HeapDumpOnOutOfMemoryError
"""
```

These configurations and strategies ensure:
- Efficient resource utilization
- Proper load distribution
- Scalable architecture
- Reliable build environment
- Easy maintenance
