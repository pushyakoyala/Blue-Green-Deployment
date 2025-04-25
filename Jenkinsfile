pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: jnlp
    image: jenkins/inbound-agent:latest
    tty: true
  - name: docker
    image: docker:24.0.6-cli
    command: ['cat']
    tty: true
    volumeMounts:
    - mountPath: /var/run/docker.sock
      name: docker-sock
  - name: trivy
    image: aquasec/trivy:latest
    command: ['cat']
    tty: true
  - name: maven
    image: maven:3.9.6-eclipse-temurin-17
    command: ['cat']
    tty: true
  - name: gke
    image: google/cloud-sdk:latest
    command: ['cat']
    tty: true
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
"""
        }
    }

    environment {
        IMAGE_NAME = "pushyakoyala/bankapp"
        KUBE_NAMESPACE = "webapps"
        SCANNER_HOME = tool 'sonar-scanner'
        KUBE_CREDENTIALS = 'k8-token'
        KUBE_CLUSTER = 'cluster-gke-demo'
        KUBE_SERVER = 'https://35.192.160.141'
        CLOUDSDK_CORE_PROJECT = 'ultimate-opus-457807-v8'
    }

    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['blue', 'green'], description: 'Choose environment to deploy')
        choice(name: 'DOCKER_TAG', choices: ['blue', 'green'], description: 'Choose the Docker image tag for the deployment')
        booleanParam(name: 'SWITCH_TRAFFIC', defaultValue: false, description: 'Switch traffic to the new env')
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-token', url: 'https://github.com/pushyakoyala/Blue-Green-Deployment.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                container('maven') {
                    withSonarQubeEnv('sonar') {
                        sh "mvn clean compile sonar:sonar -Dsonar.projectKey=nodejsmysql -Dsonar.projectName=nodejsmysql"
                    }
                }
            }
        }

        stage('Trivy FS Scan') {
            steps {
                container('trivy') {
                    sh "trivy fs --format table -o fs.html ."
                }
            }
        }

        stage('Build JAR') {
            steps {
                container('maven') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                container('docker') {
                    withDockerRegistry([credentialsId: 'docker-cred', url: 'https://index.docker.io/v1/']) {
                        sh """
                            docker build -t ${IMAGE_NAME}:${params.DOCKER_TAG} .
                            docker push ${IMAGE_NAME}:${params.DOCKER_TAG}
                        """
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                container('trivy') {
                    sh "trivy image --format table -o image.html ${IMAGE_NAME}:${params.DOCKER_TAG}"
                }
            }
        }

        stage('Gcloud Auth & Context Setup') {
            steps {
                container('gke') {
                    withCredentials([file(credentialsId: 'gcloud-creds', variable: 'GCLOUD_KEY')]) {
                        sh """
                            gcloud auth activate-service-account --key-file=${GCLOUD_KEY}
                            gcloud config set project ${CLOUDSDK_CORE_PROJECT}
                            gcloud config set compute/zone us-central1-a
                            gcloud container clusters get-credentials ${KUBE_CLUSTER} --zone us-central1-a
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                container('gke') {
                    script {
                        def deploymentFile = params.DEPLOY_ENV == 'blue' ? 'app-deployment-blue.yml' : 'app-deployment-green.yml'
                        sh """
                            kubectl apply -f mysql-ds.yml -n ${KUBE_NAMESPACE}
                            if ! kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}; then
                                kubectl apply -f bankapp-service.yml -n ${KUBE_NAMESPACE}
                            fi
                            kubectl apply -f ${deploymentFile} -n ${KUBE_NAMESPACE}
                        """
                    }
                }
            }
        }

        stage('Switch Traffic') {
            when {
                expression { return params.SWITCH_TRAFFIC }
            }
            steps {
                container('gke') {
                    script {
                        sh """
                            kubectl rollout status deployment bankapp-${params.DEPLOY_ENV} -n ${KUBE_NAMESPACE}
                            kubectl patch service bankapp-service -p \
                            '{ "spec": { "selector": { "app": "bankapp", "version": "${params.DEPLOY_ENV}" } } }' \
                            -n ${KUBE_NAMESPACE}
                        """
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                container('gke') {
                    script {
                        sh """
                            kubectl get pods -l version=${params.DEPLOY_ENV} -n ${KUBE_NAMESPACE}
                            kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: '*.html', allowEmptyArchive: true
        }
    }
}
