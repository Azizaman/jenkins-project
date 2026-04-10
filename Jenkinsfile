pipeline {
    agent any

    environment {
        App_Version = "v1.0.${BUILD_NUMBER}"
        SONARQUBE_SERVER = "sonarqube-server"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/Azizaman/datastore-project-devops.git'
            }
        }

        stage('Maven Build') {
            steps {
                sh '''
                echo "-------- Building Application --------"
                mvn clean package
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    sh '''
                    echo "-------- Running SonarQube Analysis --------"
                    mvn sonar:sonar \
                    -Dsonar.projectKey=datastore \
                    -Dsonar.host.url=http://<EC2-IP>:9000 \
                    -Dsonar.login=$SONAR_TOKEN
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Maven Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Upload Artifact to S3') {
            steps {
                sh 'aws s3 cp target/*.jar s3://datastore-artefact-store-apps-jenkins/'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t datastore:${App_Version} .'
            }
        }

        stage('Scan Docker Image') {
            steps {
                sh 'trivy image datastore:${App_Version}'
            }
        }

        stage('Tag Docker Image') {
            steps {
                sh 'docker tag datastore:${App_Version} 8072388539/datastore:${App_Version}'
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                    docker login -u $DOCKER_USER -p $DOCKER_PASS
                    docker push 8072388539/datastore:${App_Version}
                    '''
                }
            }
        }

        stage('Cleanup') {
            steps {
                sh 'docker image prune -a -f'
            }
        }

        stage('Deployment Approval') {
            steps {
                input message: "Do you want to deploy?"
            }
        }

        stage('Trigger Deployment Job') {
            steps {
                build job: "KubernetesDeployment",
                parameters: [
                    string(name: "App_Name", value: "datastore-deploy"),
                    string(name: "App_Version", value: "${App_Version}")
                ]
            }
        }
    }
}
