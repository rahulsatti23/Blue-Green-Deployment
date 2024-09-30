pipeline {
    agent any

    tools {
        nodejs 'node21'
        maven 'maven3' // Assuming Maven is configured in Jenkins
    }

    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['blue', 'green'], description: 'Choose which environment to deploy: Blue or Green')
        choice(name: 'DOCKER_TAG', choices: ['blue', 'green'], description: 'Choose the Docker image tag for the deployment')
        booleanParam(name: 'SWITCH_TRAFFIC', defaultValue: false, description: 'Switch traffic between Blue and Green')
    }

    environment {
        IMAGE_NAME = "rahulsatti/bankapp"
        TAG = "${params.DOCKER_TAG}"
        KUBE_NAMESPACE = 'webapps'
        SONARQUBE_SERVER = 'sonar' // Name of your SonarQube server in Jenkins configuration
        SONAR_TOKEN = credentials('sonar-token') // Replace with your actual credential ID
        DOCKER_CRED = credentials('docker-cred') // Replace with your Docker credential ID
        K8_TOKEN = credentials('k8-token') // Replace with your Kubernetes token ID
        MYSQL_ROOT_PASSWORD = credentials('mysql-root-password') // Replace with your MySQL root password credential ID
        MYSQL_DATABASE = "bankappdb"
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/rahulsatti23/Blue-Green-Deployment.git'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
               withSonarQubeEnv('sonar')  {
                    sh 'mvn clean verify sonar:sonar'
                }
            }
        }
        
        stage('Trivy FS Scan') {
            steps {
                sh "trivy fs --format table -o fs.html ."
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t ${IMAGE_NAME}:${TAG} ."
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh "trivy image --format table -o image.html ${IMAGE_NAME}:${TAG}"
            }
        }

        stage('Docker Push Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker push ${IMAGE_NAME}:${TAG}"
                    }
                }
            }
        }

        stage('Deploy MySQL Deployment and Service') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'rahul-cluster', credentialsId: 'k8-token', namespace: KUBE_NAMESPACE, serverUrl: 'https://5ECE6B22A18DBC01D2C2727C22408335.gr7.ap-south-1.eks.amazonaws.com') {
                        sh """
                        kubectl apply -f mysql-ds.yml -n ${KUBE_NAMESPACE}
                        kubectl set env deployment/mysql MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD} MYSQL_DATABASE=${MYSQL_DATABASE} -n ${KUBE_NAMESPACE}
                        """
                    }
                }
            }
        }

        stage('Deploy Service App') {
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'rahul-cluster', credentialsId: 'k8-token', namespace: KUBE_NAMESPACE, serverUrl: 'https://5ECE6B22A18DBC01D2C2727C22408335.gr7.ap-south-1.eks.amazonaws.com') {
                        sh """
                        if ! kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}; then
                            kubectl apply -f bankapp-service.yml -n ${KUBE_NAMESPACE}
                        fi
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    def deploymentFile = params.DEPLOY_ENV == 'blue' ? 'app-deployment-blue.yml' : 'app-deployment-green.yml'
                    withKubeConfig(caCertificate: '', clusterName: 'rahul-cluster', credentialsId: 'k8-token', namespace: KUBE_NAMESPACE, serverUrl: 'https://5ECE6B22A18DBC01D2C2727C22408335.gr7.ap-south-1.eks.amazonaws.com') {
                        sh "kubectl apply -f ${deploymentFile} -n ${KUBE_NAMESPACE}"
                    }
                }
            }
        }

        stage('Switch Traffic Between Blue & Green Environment') {
            when {
                expression { params.SWITCH_TRAFFIC }
            }
            steps {
                script {
                    def newEnv = params.DEPLOY_ENV
                    withKubeConfig(caCertificate: '', clusterName: 'rahul-cluster', credentialsId: 'k8-token', namespace: KUBE_NAMESPACE, serverUrl: 'https://5ECE6B22A18DBC01D2C2727C22408335.gr7.ap-south-1.eks.amazonaws.com') {
                        sh '''
                            kubectl patch service bankapp-service -p '{"spec": {"selector": {"app": "bankapp", "version": "'"${newEnv}"'"}}}' -n ${KUBE_NAMESPACE}
                        '''
                    }
                    echo "Traffic has been switched to the ${newEnv} environment."
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    def verifyEnv = params.DEPLOY_ENV
                    withKubeConfig(caCertificate: '', clusterName: 'rahul-cluster', credentialsId: 'k8-token', namespace: KUBE_NAMESPACE, serverUrl: 'https://5ECE6B22A18DBC01D2C2727C22408335.gr7.ap-south-1.eks.amazonaws.com') {
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
