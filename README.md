# README: Jenkins Pipeline Setup for Java/Maven CI/CD

This README provides a guide to setting up a Jenkins pipeline for building, testing, deploying, and managing Java/Maven projects. It covers everything from retrieving versions from Artifactory to establishing VPN connections and deploying to cloud services.

---

## Overview

The pipeline automates the following tasks:

1. Building and testing Java/Maven applications.
2. Analyzing code quality using SonarQube.
3. Managing artifacts with JFrog Artifactory.
4. Deploying applications to an AWS EC2 instance as a systemd service.

The workflow is designed to be project-agnostic, dynamic, and easy to maintain.

---

## Pipeline Workflow

### Trigger

- A Git commit triggers the pipeline.

### Stages

1. **Build Stage**: Compiles and packages the application using Maven.
2. **Test Stage**: Executes unit tests and collects reports.
3. **Code Quality Stage**: Uses SonarQube to perform static code analysis.
4. **Artifact Deployment**: Uploads built artifacts (e.g., JAR files) to JFrog Artifactory.
5. **Deployment Stage**: Deploys the artifact to an AWS EC2 instance.
6. **VPN Establishment**: Sets up a secure VPN connection before deployment (if required).

---

## Key Components

### 1. **Artifactory Integration**

#### Fetching Versions

The pipeline dynamically retrieves available artifact versions from Artifactory. Example:

```groovy
stage('Fetch Versions') {
    steps {
        script {
            def pom = readMavenPom file: 'pom.xml'
            def artifactId = pom.artifactId

            def versions = sh(
                script: '''
                curl -s -H "X-JFrog-Art-Api: ${ARTIFACTORY_API_KEY}" \
                ${ARTIFACTORY_URL}/api/storage/${REPOSITORY}/${GROUP_PATH}/${artifactId} \
                | grep '"uri"' \
                | grep -Eo '/[0-9]+\\.[0-9]+\\.[0-9]+' \
                | sed 's|^/||'
                ''',
                returnStdout: true
            ).trim()

            echo "Artifact: ${artifactId}"
            echo "Versions: ${versions}"

            writeFile file: 'versions.txt', text: versions
        }
    }
}
```

#### Uploading Artifacts

Artifacts are uploaded using the following command:

```groovy
sh '''
    curl -s -H "X-JFrog-Art-Api: ${ARTIFACTORY_API_KEY}" \
    -T target/${artifactId}-${version}.jar \
    ${ARTIFACTORY_URL}/${REPOSITORY}/${GROUP_PATH}/${artifactId}/${version}/${artifactId}-${version}.jar
'''
```

---

### 2. **Dynamic Parameterization**

The pipeline allows selecting versions dynamically via a dropdown menu:

```groovy
parameters {
    choice(
        name: 'VERSION',
        choices: readFile('versions.txt').trim(),
        description: 'Select the version of the JAR to deploy'
    )
}
```

---

### 3. **VPN Connection Management**

#### Optimized VPN Connection Setup

The pipeline establishes a VPN connection dynamically and pings the target server to verify connectivity:

```groovy
stage('Establish VPN Connection') {
    steps {
        script {
            sh '''
            set -e

            openvpn --config ${VPN_CONFIG_FILE} --daemon --log /tmp/openvpn.log

            for i in {1..3}; do
                echo "Waiting for VPN to establish... Attempt $i"
                sleep 5
                if ping -c 1 ${TARGET_IP} > /dev/null 2>&1; then
                    echo "VPN connection successfully established!"
                    exit 0
                fi
            done

            echo "ERROR: VPN connection could not be established after 3 attempts."
            exit 1
            '''
        }
    }
}
```

---

### 4. **Deployment to AWS EC2**

Artifacts are deployed as systemd services on EC2 instances:

```groovy
stage('Deploy to Server') {
    steps {
        script {
            sh """
            scp -i ${SSH_KEY} ./target/${artifactId}-${VERSION}.jar ec2-user@${REMOTE_SERVER}:/opt/${artifactId}/latest/

            ssh -i ${SSH_KEY} ec2-user@${REMOTE_SERVER} << EOF
                set -e

                sudo mkdir -p /opt/${artifactId}/latest
                sudo mkdir -p /opt/${artifactId}/current

                sudo ln -sfn /opt/${artifactId}/latest/${artifactId}-${VERSION}.jar /opt/${artifactId}/current/${artifactId}.jar.new
                sudo mv -f /opt/${artifactId}/current/${artifactId}.jar.new /opt/${artifactId}/current/${artifactId}.jar

                if [ -L /opt/${artifactId}/current/${artifactId}.jar ]; then
                    sudo systemctl restart ${artifactId}
                else
                    echo "ERROR: Symlink replacement failed!"
                    exit 1
                fi
            EOF
            """
        }
    }
}
```

---

## Troubleshooting

1. **Dynamic Variables Not Interpolating**:

   - Ensure that Groovy variables are correctly passed using `"""` for interpolation or concatenated explicitly.

2. **Artifactory API Errors**:

   - Verify API permissions and URL structure.
   - Ensure the `ARTIFACTORY_API_KEY` has sufficient privileges.

3. **VPN Connection Fails**:

   - Check the VPN configuration file and logs (`/tmp/openvpn.log`).
   - Ensure the target server is reachable once the VPN is established.

4. **Deployment Issues**:

   - Verify the systemd service is correctly configured on the target EC2 instance.
   - Check file permissions and paths during deployment.

---

## Additional Notes

1. **Tools Used**:

   - Maven for building and testing.
   - SonarQube for code quality.
   - JFrog Artifactory for artifact management.
   - AWS EC2 for deployment.

2. **Environment Variables**:

   - `ARTIFACTORY_URL`: Base URL of the Artifactory server.
   - `ARTIFACTORY_API_KEY`: API key for Artifactory authentication.
   - `VPN_CONFIG_FILE`: Configuration file for OpenVPN.
   - `SSH_KEY`: SSH private key for EC2 access.

3. **Extensibility**:

   - Additional stages can be added for integration tests, canary deployments, etc.

---

This pipeline is designed to be adaptable to various Java/Maven projects while ensuring simplicity and maintainability.
