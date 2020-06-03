pipeline {
  agent {
    kubernetes {
      yamlFile 'build-agent.yaml'
      defaultContainer 'maven'
      idleMinutes 1
    }
  }
  stages {
    stage('Setup') {
      parallel {
        stage('Install Dependencies') {
          steps {
            container('maven') {
              sh 'mvn install -DskipTests -Dspotbugs.skip=true -Ddependency-check.skip=true'
            }
          }
        }
        stage('Secrets scanner') {
          steps {
            container('trufflehog') {
              sh 'git clone ${GIT_URL}'
              sh 'cd secure-pipeline-java-demo && ls -al'
              sh 'cd secure-pipeline-java-demo && trufflehog .'
              sh 'rm -rf secure-pipeline-java-demo'
            }
          }
        }
      }
    }
    stage('Static Analysis') {
      parallel {
        stage('Unit Tests') {
          steps {
            container('maven') {
              sh 'mvn test'
            }
          }
        }
        stage('Dependency Checker') {
          steps {
            container('maven') {
              sh 'mvn org.owasp:dependency-check-maven:check'
            }
          }
          post {
            always {
              archiveArtifacts allowEmptyArchive: true, artifacts: 'target/dependency-check-report.html', fingerprint: true, onlyIfSuccessful: true
            }
          }
        }
        stage('Spot Bugs - Security') {
          steps {
            container('maven') {
              sh 'mvn compile spotbugs:check'
            }
          }
        }
        stage('OSS License Checker') {
          steps {
            container('licensefinder') {
              sh 'ls -al'
              sh '''#!/bin/bash --login
                      /bin/bash --login
                      rvm use default
                      gem install license_finder
                      license_finder
                    '''
            }
          }
        }
        stage('SCA') {
          steps {
            container('maven') {
              sh 'mvn org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom'
            }
          }
          post {
            success {
              dependencyTrackPublisher artifact: 'target/bom.xml', projectId: '77871267-ae32-447f-8ea1-cc4b0e3145ad', synchronous: false
              archiveArtifacts allowEmptyArchive: true, artifacts: 'target/bom.xml', fingerprint: true, onlyIfSuccessful: true
            }
          }
        }
      }
    }
    stage('Build') {
      steps {
        container('maven') {
          sh 'mvn package'
        }
        container('docker-cmds') {
          sh 'ls -al'
          sh 'docker build . -t sample-app'
        }
      }
    }
    stage('Artefact Analysis') {
      parallel {
        stage('Image Scan') {
          steps {
            container('docker-cmds') {
              sh '''#!/bin/sh
                    apk add --update-cache --upgrade curl rpm
                    export TRIVY_VERSION="0.8.0"
                    echo $TRIVY_VERSION
                    wget https://github.com/aquasecurity/trivy/releases/download/v${TRIVY_VERSION}/trivy_${TRIVY_VERSION}_Linux-64bit.tar.gz
                    tar zxvf trivy_${TRIVY_VERSION}_Linux-64bit.tar.gz
                    mv trivy /usr/local/bin
                    trivy --cache-dir /tmp/trivycache/ sample-app:latest
                  '''
              }
          }
        }
        stage('Image Hardening') {
          steps {
            container('docker-cmds') {
              sh 'docker run --rm -v /var/run/docker.sock:/var/run/docker.sock goodwithtech/dockle sample-app:latest'
            }
          }
        }
        stage('K8s Hardening') {
          steps {
            container('docker-cmds') {
              sh 'docker run -i kubesec/kubesec:512c5e0 scan /dev/stdin < pod.yaml'
            }
          }
        }
      }
    }
  }
}
