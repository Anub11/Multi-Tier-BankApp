pipeline {
    agent any

    tools {
        maven "maven3"
    }

     parameters {
        choice(name: 'PARAM_ENV', choices: ['dev', 'qa'], description: 'Select the environment to deploy')
        string(name: 'IMAGE_TAG', description: 'Docker image tag to deploy')
    }
    
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Git checkout') {
            steps {
                checkout scm
            }
        }
        stage('Set Image Tag') {
            steps {
                script {
                    env.FINAL_IMAGE_TAG = params.IMAGE_TAG?.trim()
                        ? params.IMAGE_TAG.trim()
                        : "1.0.${BUILD_NUMBER}"
                    echo "Using Docker image tag: ${env.FINAL_IMAGE_TAG}"
                }
            }
        }
        

        stage('Compile') {
            when { expression { params.PARAM_ENV == 'dev' } }
            steps {
                sh 'mvn compile'
            }
        }

        stage('Test') {
            when { expression { params.PARAM_ENV == 'dev' } }
            steps {
                sh 'mvn test'
            }
        }

        stage('Run Gitleaks') {
            when { expression { params.PARAM_ENV == 'dev' } }
            steps {
                sh 'gitleaks detect --source . --report-path gitleaks-report.json || true'
            }
        }

        stage('Run Trivy') {
            when { expression { params.PARAM_ENV == 'dev' } }
            steps {
                sh 'trivy fs --exit-code 0 --severity HIGH,CRITICAL --format json -o trivy-fs-report.json .'
            }
        }
        
        stage('SonarQube Analysis') {
            when { expression { params.PARAM_ENV == 'dev' } }
            steps {
                withSonarQubeEnv('sonar') {
                     sh '''$SCANNER_HOME/bin/sonar-scanner \
                     -Dsonar.projectName=bankapp \
                     -Dsonar.projectKey=bankapp \
                     -Dsonar.java.binaries=. '''
                     
                }
            }
        }
        
        stage('Quality Gate') {
            when { expression { params.PARAM_ENV == 'dev' } }
            steps {
                script{
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar'
                }
            }
        }

        stage('Build') {
            when { expression { params.PARAM_ENV == 'dev' } }
            steps {
                sh 'mvn package'
            }
        }
        
        stage('Publish Artifact') {
            when { expression { params.PARAM_ENV == 'dev' } }
            steps {
                script {
                    env.ARTIFACT_VERSION = "1.0.${BUILD_NUMBER}-SNAPSHOT"
                }
                withMaven(globalMavenSettingsConfig: '8489adb4-269c-4cde-994a-d729f43ead3e', jdk: '', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh "mvn deploy -Drevision=${ARTIFACT_VERSION}"
                }   
            }
        }
        
        stage('Build and tag docker image') {
            when { expression { params.PARAM_ENV == 'dev' } }
            steps {
                script{
                    withDockerRegistry(credentialsId: 'docker') {
                        sh "echo ${params.IMAGE_TAG}"
                        sh "docker build -t anub11/bankapp:${env.FINAL_IMAGE_TAG} ."
                    }
                }
            }
        }
        
        stage('Docker image scan') {
            when { expression { params.PARAM_ENV == 'dev' } }
            steps {
                sh "trivy image --exit-code 0 --severity HIGH,CRITICAL --format json -o trivy-image-report.json anub11/bankapp:${env.FINAL_IMAGE_TAG}"
            }
        }
        
        stage('Push docker image') {
            when { expression { params.PARAM_ENV == 'dev' } }
            steps {
                script{
                    withDockerRegistry(credentialsId: 'docker') {
                        sh "docker push anub11/bankapp:${env.FINAL_IMAGE_TAG}"
                    }
                }
            }
        }
        
        stage('Deploy to EKS') {
            steps {
                sh 'ls -l'   
                sh "sed -i 's|image:.*|image: anub11/bankapp:${env.FINAL_IMAGE_TAG}|' ds.yml"
                sh "echo deployed!!"
            }
        }
        
        stage('Verify deployment EKS') {
            steps {
                echo "Using Docker image tag: ${env.FINAL_IMAGE_TAG}"
            }
        }
        
        
    }
}
