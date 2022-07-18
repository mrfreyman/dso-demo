pipeline {
  
  environment {
  //not working  ARGO_SERVER = '44.202.193.149:32100'
 //also not working   ARGO_SERVER = '172.31.13.142:32100'
    ARGO_SERVER = 'localhost:32100'
  }
    
  agent {
    kubernetes {
      yamlFile 'build-agent.yaml'
      defaultContainer 'maven'
      idleMinutes 1
    }
  }
  
  stages {
    
    stage('Build') {
      parallel {
        stage('MVN Compile') {
          steps {
            container('maven') {
              sh 'mvn compile'
            }
          }
        }
      }
    }
    
    stage('Static Analysis') {
      parallel {
        
        stage('MVN Unit Tests') {
          steps {
            container('maven') {
              sh 'mvn test'
            }
          }
        }
        
        stage('OWASP Dep Check SCA') {
          steps {
            container('maven') {
              catchError(buildResult: 'SUCCESS', stageResult:'FAILURE') {
                sh 'mvn org.owasp:dependency-check-maven:check'
              }
            }
          }
          post {
            always {
              archiveArtifacts allowEmptyArchive: true,artifacts: 'target/dependency-check-report.html', fingerprint:true, onlyIfSuccessful: true// dependencyCheckPublisher pattern: 'report.xml'
            }
          }
        }  
       
        stage('License_Finder OSS') {
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
        
      }
    }
    
    
   stage('slSCAN SAST') {
          steps {
            container('slscan') {
              sh 'scan --type java,depscan --build'
            }
          }
          post {
            success {
              archiveArtifacts allowEmptyArchive: true,artifacts: 'reports/*', fingerprint: true, onlyIfSuccessful:true
            }
          }
      }
   
    
    stage('Package') {
      parallel {
        stage('MVN Create Jarfile') {
          steps {
            container('maven') {
              sh 'mvn package -DskipTests'
            }
          }
        }
        stage('Kaniko OCI Image BnP') {
          steps {
            container('kaniko') {
              sh '/kaniko/executor -f `pwd`/Dockerfile -c `pwd` --insecure --skip-tls-verify --cache=true --destination=docker.io/mrfreyman/dso-demo'
            }
           }
         }      
       }
     }
    
    
    stage('Image Analysis') {
          parallel {
            stage('Dockle Image Linting') {
              steps {
                container('docker-tools') {
                  sh 'dockle docker.io/mrfreyman/dso-demo'
                }
              }
            }
            stage('Trivy Image Scan') {
              steps {
                container('docker-tools') {
                  sh 'trivy image --exit-code 1 mrfreyman/dso-demo'
                }
              }
            }
          }
      }
    
    
    stage('Argocd Deploy to Dev') {
      environment {
        AUTH_TOKEN = credentials('argocd-jenkins-deployer-token')
      }
      steps {
        container('docker-tools') {
          sh 'docker run -t schoolofdevops/argocd-cli argocd app sync dso-demo  --insecure --server $ARGO_SERVER --auth-token $AUTH_TOKEN'
          sh 'docker run -t schoolofdevops/argocd-cli argocd app wait dso-demo --health --timeout 300   --insecure --server $ARGO_SERVER --auth-token $AUTH_TOKEN'
        }
      }
    }
    
  }
}
