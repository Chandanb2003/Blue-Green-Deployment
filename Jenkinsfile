pipeline {
    agent any
    
    tools {
        maven 'maven3'
    }
    
    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['blue', 'green'], description: 'Choose which environment to deploy: Blue or Green')
        choice(name: 'DOCKER_TAG', choices: ['blue', 'green'], description: 'Choose the Docker image tag for the deployment')
        booleanParam(name: 'SWITCH_TRAFFIC', defaultValue: false, description: 'Switch traffic between Blue and Green')
    }
    
    environment {
        IMAGE_NAME = "chandan669/bankapp"
        TAG = "${params.DOCKER_TAG}"
        KUBE_NAMESPACE = 'webapps'
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'github-token', url: 'https://github.com/Chandanb2003/Blue-Green-Deployment.git'
            }
        }
        
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('Tests') {
            steps {
                sh "mvn clean test -X -DskipTests=true"
            }
        }
        
        stage('Trivy FS scan') {
            steps {
                sh "trivy fs --format table -o fs.html ."
            }
        }
        
        stage('Sonarqube analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh "$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectKey=BlueGreen -Dsonar.projectName=BlueGreen -Dsonar.java.binaries=target"
                }
            }
        }
        
        stage('Quality gate check') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }
        
        stage('Build') {
            steps {
                sh "mvn package -DskipTests=true"
            }
        }
        
        stage('Publish artifact To Nexus') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'nexus-creds',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASSWORD'
                    )
                ]) {
                    withMaven(
                        maven: 'maven3',
                        mavenSettingsConfig: '22894c1a-0918-4cf0-80e2-cc90761c7255',
                        traceability: true
                    ) {
                        sh 'mvn clean deploy -DskipTests'
                    }
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', url: '') {
                        sh "docker build -t ${IMAGE_NAME}:${TAG} ."
                        sh "docker push ${IMAGE_NAME}:${TAG}"
                    }
                }
            }
        }

        stage('Authenticate to GKE') {
            steps {
                withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GCP_KEY')]) {
                    sh '''
                        gcloud auth activate-service-account --key-file=$GCP_KEY
                        gcloud config set project model-deployments
                        gcloud config set compute/region us-central1
                        gcloud container clusters get-credentials cluster-1
                    '''
                }
            }
        }

        stage('Deploy MySQL Deployment and Service') {
            steps {
                withKubeConfig(credentialsId: 'gke-kubeconfig') {
                    sh '''
                        kubectl get nodes
                        kubectl apply -f mysql-ds.yml -n ${KUBE_NAMESPACE}
                    '''
                }
            }
        }
        
        stage('Deploy SVC app') {
            steps {
                withKubeConfig(credentialsId: 'gke-kubeconfig') {
                    sh '''
                        if ! kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}; then
                            kubectl apply -f bankapp-service.yml -n ${KUBE_NAMESPACE}
                        fi
                    '''
                }
            }
        }
        
        stage('Deploy Application') {
            steps {
                script {
                    def deploymentFile = params.DEPLOY_ENV == 'blue' ? 'app-deployment-blue.yml' : 'app-deployment-green.yml'
                    withKubeConfig(credentialsId: 'gke-kubeconfig') {
                        sh "kubectl apply -f ${deploymentFile} -n ${KUBE_NAMESPACE}"
                    }
                }
            }
        }

        stage('Switch Traffic Between Blue & Green Environment') {
            when {
                expression { return params.SWITCH_TRAFFIC }
            }
            steps {
                script {
                    def newEnv = params.DEPLOY_ENV
                    withKubeConfig(credentialsId: 'gke-kubeconfig') {
                        sh """
                            kubectl patch service bankapp-service -p '{ "spec": { "selector": { "app": "bankapp", "version": "${newEnv}" }}}' -n ${KUBE_NAMESPACE}
                        """
                    }
                    echo "Traffic has been switched to the ${newEnv} environment."
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    def verifyEnv = params.DEPLOY_ENV
                    withKubeConfig(credentialsId: 'gke-kubeconfig') {
                        sh """
                            kubectl get pods -l version=${verifyEnv} -n ${KUBE_NAMESPACE}
                            kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}
                        """
                    }
                }
            }
        }
    }
}
