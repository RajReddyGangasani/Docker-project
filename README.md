# Docker-project

pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'mysonar'
    }
    stages {
        stage ("cleanWs") {
            steps {
                cleanWs()
            }
        }
        stage ("Code") {
            steps {
                git branch: 'main', url: 'https://github.com/RajReddyGangasani/Zomato-Repo.git'
            }
        }
        stage ("CQA") {
            steps {
                withSonarQubeEnv('mysonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=zomato \
                    -Dsonar.projectKey=zomato ''' 
                }
            }
        }
        stage ("quality gates") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        stage ("build") {
            steps{
                sh 'npm install'
            }
        }
        stage ("OWASP") {
          steps {
              dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
              dependencyCheckPublisher pattern: '**dependency-check-report.xml'
          }  
        }
        stage ("Dockerfile") {
            steps {
                sh 'docker build -t rajgangasani/raj-nav:zomato .'
            }
        }
        stage ("TRIVY FS SCAN)") {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage ("TRIVY imagescan") {
            steps {
                sh 'trivy image rajgangasani/raj-nav:zomato'
            }
        }
        stage ("Docker Registery") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub') {
                       sh 'docker push rajgangasani/raj-nav:zomato'
                    }
                }
            }
        }
        stage ("docker contianer") {
            steps {
                sh 'docker run -itd --name cont2 -p 3000:3000 rajgangasani/raj-nav:zomato'
            }
        }
    }
}
