pipeline {
    agent any

    tools {
        maven 'maven3'
    }

    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['blue', 'green'], description: 'Deploy environment')
        choice(name: 'DOCKER_TAG', choices: ['blue', 'green'], description: 'Docker image tag')
        booleanParam(name: 'SWITCH_TRAFFIC', defaultValue: false, description: 'Switch traffic')
    }

    environment {
        IMAGE_NAME = "chandan669/bankapp"
        TAG = "${params.DOCKER_TAG}"
        KUBE_NAMESPACE = "webapps"
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: 'main',
                    credentialsId: 'github-token',
                    url: 'https://github.com/Chandanb2003/Blue-Green-Deployment.git'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Tests') {
            steps {
                sh 'mvn clean test -DskipTests=true'
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh 'trivy fs --format table -o fs.html .'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectKey=BlueGreen \
                    -Dsonar.projectName=BlueGreen \
                    -Dsonar.java.binaries=target
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn package -DskipTests=true'
            }
        }

        stage('Publish Artifact to Nexus') {
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
                        mavenSettingsConfig: '22894c1a-0918-4cf0-80e2-cc90761c7255'
                    ) {
                        sh 'mvn clean deploy -DskipTests'
                    }
                }
            }
        }

        stage('Docker Build') {
            steps {
                withDockerRegistry(
                    credentialsId: 'docker-cred',
                    url: 'https://index.docker.io/v1/'
                ) {
                    sh "docker build -t ${IMAGE_NAME}:${TAG} ."
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh "trivy image ${IMAGE_NAME}:${TAG}"
            }
        }

        stage('Docker Push') {
            steps {
                withDockerRegistry(
                    credentialsId: 'docker-cred',
                    url: 'https://index.docker.io/v1/'
                ) {
                    sh "docker push ${IMAGE_NAME}:${TAG}"
                }
            }
        }

        stage('Deploy MySQL') {
            steps {
                withKubeConfig(credentialsId: 'gke-kubeconfig', namespace: "${KUBE_NAMESPACE}") {
                    sh 'kubectl apply -f mysql-ds.yml'
                }
            }
        }

        stage('Deploy Service') {
            steps {
                withKubeConfig(credentialsId: 'gke-kubeconfig', namespace: "${KUBE_NAMESPACE}") {
                    sh '''
                    if ! kubectl get svc bankapp-service; then
                        kubectl apply -f bankapp-service.yml
                    fi
                    '''
                }
            }
        }

        stage('Deploy Application') {
            steps {
                script {
                    def deploymentFile = params.DEPLOY_ENV == 'blue'
                        ? 'app-deployment-blue.yml'
                        : 'app-deployment-green.yml'

                    withKubeConfig(credentialsId: 'gke-kubeconfig', namespace: "${KUBE_NAMESPACE}") {
                        sh "kubectl apply -f ${deploymentFile}"
                    }
                }
            }
        }

        stage('Switch Traffic') {
            when {
                expression { params.SWITCH_TRAFFIC }
            }
            steps {
                withKubeConfig(credentialsId: 'gke-kubeconfig', namespace: "${KUBE_NAMESPACE}") {
                    sh """
                    kubectl patch service bankapp-service \
                    -p '{"spec":{"selector":{"app":"bankapp","version":"${params.DEPLOY_ENV}"}}}'
                    """
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withKubeConfig(credentialsId: 'gke-kubeconfig', namespace: "${KUBE_NAMESPACE}") {
                    sh '''
                    kubectl get pods -o wide
                    kubectl get svc bankapp-service
                    '''
                }
            }
        }
    }
}
