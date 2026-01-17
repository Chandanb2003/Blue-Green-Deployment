pipeline {
    agent any

    tools {
        maven 'maven3'
    }

    parameters {
        choice(
            name: 'DEPLOY_ENV',
            choices: ['blue', 'green'],
            description: 'Choose environment to deploy'
        )
        choice(
            name: 'DOCKER_TAG',
            choices: ['blue', 'green'],
            description: 'Docker image tag'
        )
        booleanParam(
            name: 'SWITCH_TRAFFIC',
            defaultValue: false,
            description: 'Switch traffic to selected environment'
        )
    }

    environment {
        IMAGE_NAME     = "chandan669/bankapp"
        TAG            = "${params.DOCKER_TAG}"
        KUBE_NAMESPACE = "webapps"
        SCANNER_HOME   = tool 'sonar-scanner'
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
                sh 'mvn test -DskipTests=true'
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
                        ${SCANNER_HOME}/bin/sonar-scanner \
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

        stage('Build Package') {
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
                        sh 'mvn deploy -DskipTests'
                    }
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                withDockerRegistry(credentialsId: 'docker-cred', url: '') {
                    sh '''
                        docker build -t ${IMAGE_NAME}:${TAG} .
                        docker push ${IMAGE_NAME}:${TAG}
                    '''
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
                        kubectl get nodes
                    '''
                }
            }
        }

        stage('Deploy MySQL Deployment and Service') {
            steps {
                sh '''
                    kubectl apply -f mysql-ds.yml -n ${KUBE_NAMESPACE}
                '''
            }
        }

        stage('Deploy Application Service') {
            steps {
                sh '''
                    kubectl get svc bankapp-service -n ${KUBE_NAMESPACE} \
                    || kubectl apply -f bankapp-service.yml -n ${KUBE_NAMESPACE}
                '''
            }
        }

        stage('Deploy Application (Blue/Green)') {
            steps {
                script {
                    def deploymentFile =
                        params.DEPLOY_ENV == 'blue'
                        ? 'app-deployment-blue.yml'
                        : 'app-deployment-green.yml'

                    sh """
                        kubectl apply -f ${deploymentFile} -n ${KUBE_NAMESPACE}
                    """
                }
            }
        }

        stage('Switch Traffic') {
            when {
                expression { params.SWITCH_TRAFFIC }
            }
            steps {
                sh """
                    kubectl patch svc bankapp-service \
                    -n ${KUBE_NAMESPACE} \
                    -p '{ "spec": { "selector": { "app": "bankapp", "version": "${params.DEPLOY_ENV}" }}}'
                """
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    kubectl get pods -n ${KUBE_NAMESPACE}
                    kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}
                '''
            }
        }
    }
}
