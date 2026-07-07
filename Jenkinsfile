pipeline {
    agent any
    
    tools {
        maven 'maven3'
    }

    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['blue', 'green'], description: 'Choose environment to deploy')
        choice(name: 'DOCKER_TAG', choices: ['blue', 'green'], description: 'Docker image tag')
        booleanParam(name: 'SWITCH_TRAFFIC', defaultValue: false, description: 'Switch traffic after deployment')
    }

    environment {
        IMAGE_NAME = "milindnagne/bankapp"
        TAG = "${params.DOCKER_TAG}"
        KUBE_NAMESPACE = "webapps"
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Milind2803/Blue-Green-Deployment.git'
            }
        }
        stage('Check Java Environment') {
    steps {
        sh '''
        echo "JAVA_HOME=$JAVA_HOME"
        which java
        java -version
        which javac
        javac -version
        mvn -version
        '''
    }
}

        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Unit Test') {
            steps {
                sh 'mvn test -DskipTests'
            }
        }

        stage('Trivy File System Scan') {
            steps {
                sh 'trivy fs --format table -o fs.html .'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                        mvn sonar:sonar \
                        -Dsonar.projectKey=nodejsmysql \
                        -Dsonar.projectName=nodejsmysql
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

        stage('Package') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Publish Artifact to Nexus') {
            steps {
                withMaven(
                    globalMavenSettingsConfig: 'maven-settings',
                    maven: 'maven3'
                ) {
                    sh 'mvn deploy -DskipTests'
                }
            }
        }

        stage('Docker Build') {
            steps {
                script {
                    withDockerRegistry(
                        credentialsId: 'docker-cred',
                        url: 'https://index.docker.io/v1/'
                    ) {
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

        stage('Docker Push') {
            steps {
                script {
                    withDockerRegistry(
                        credentialsId: 'docker-cred',
                        url: 'https://index.docker.io/v1/'
                    ) {
                        sh "docker push ${IMAGE_NAME}:${TAG}"
                    }
                }
            }
        }

        stage('Deploy MySQL') {
            steps {
                script {
                    withKubeConfig(
                        credentialsId: 'k8-token',
                        clusterName: 'devopsshack-cluster',
                        namespace: 'webapps',
                        serverUrl: 'https://91FC77F9D8A76B988BAA48A5DBC0ED9F.gr7.ap-south-1.eks.amazonaws.com'
                    ) {
                        sh "kubectl apply -f mysql-ds.yml -n ${KUBE_NAMESPACE}"
                    }
                }
            }
        }

        stage('Deploy Service') {
            steps {
                script {
                    withKubeConfig(
                        credentialsId: 'k8-token',
                        clusterName: 'devopsshack-cluster',
                        namespace: 'webapps',
                        serverUrl: 'https://91FC77F9D8A76B988BAA48A5DBC0ED9F.gr7.ap-south-1.eks.amazonaws.com'
                    ) {

                        sh """
                        if ! kubectl get svc bankapp-service -n ${KUBE_NAMESPACE}; then
                            kubectl apply -f bankapp-service.yml -n ${KUBE_NAMESPACE}
                        fi
                        """
                    }
                }
            }
        }

        stage('Deploy Application') {
            steps {
                script {

                    def deploymentFile = params.DEPLOY_ENV == "blue" ?
                            "app-deployment-blue.yml" :
                            "app-deployment-green.yml"

                    withKubeConfig(
                        credentialsId: 'k8-token',
                        clusterName: 'devopsshack-cluster',
                        namespace: 'webapps',
                        serverUrl: 'https://91FC77F9D8A76B988BAA48A5DBC0ED9F.gr7.ap-south-1.eks.amazonaws.com'
                    ) {

                        sh "kubectl apply -f ${deploymentFile} -n ${KUBE_NAMESPACE}"

                    }
                }
            }
        }

        stage('Switch Traffic') {

            when {
                expression {
                    params.SWITCH_TRAFFIC
                }
            }

            steps {

                script {

                    def newEnv = params.DEPLOY_ENV

                    withKubeConfig(
                        credentialsId: 'k8-token',
                        clusterName: 'devopsshack-cluster',
                        namespace: 'webapps',
                        serverUrl: 'https://91FC77F9D8A76B988BAA48A5DBC0ED9F.gr7.ap-south-1.eks.amazonaws.com'
                    ) {

                        sh """
                        kubectl patch service bankapp-service \
                        -p '{"spec":{"selector":{"app":"bankapp","version":"${newEnv}"}}}' \
                        -n ${KUBE_NAMESPACE}
                        """

                    }

                    echo "Traffic switched to ${newEnv}"

                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {

                    withKubeConfig(
                        credentialsId: 'k8-token',
                        clusterName: 'devopsshack-cluster',
                        namespace: 'webapps',
                        serverUrl: 'https://91FC77F9D8A76B988BAA48A5DBC0ED9F.gr7.ap-south-1.eks.amazonaws.com'
                    ) {

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

        success {
            echo "Pipeline completed successfully."
        }

        failure {
            echo "Pipeline failed."
        }

    }
}
