pipeline {
    agent any

    environment {
        registry = "oli0404/vproapp04"
        registryCredential = "dockerhub"
    }

    stages {
        stage('BUILD') {
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        stage('UNIT TEST') {
            steps {
                sh 'mvn test'
            }
        }

        stage('INTEGRATION TEST') {
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }

        stage('CODE ANALYSIS WITH CHECKSTYLE') {
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }

        stage('CODE ANALYSIS with SONARCLOUD') {
            steps {
                withSonarCloudEnv('sonar-cloud-pro') {
                    sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                    -Dsonar.projectName=vprofile-repo \
                    -Dsonar.projectVersion=1.0 \
                    -Dsonar.sources=src/ \
                    -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                    -Dsonar.junit.reportsPath=target/surefire-reports/ \
                    -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                    -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml \
                    -Dsonar.organization=your-org-name \
                    -Dsonar.host.url=https://sonarcloud.io \
                    -Dsonar.login=${env.SONAR_TOKEN}''' // Use an environment variable for the token
                }

                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build App Image') {
            steps {
                script {
                    dockerImage = docker.build "${registry}:V${BUILD_NUMBER}"
                }
            }
        }

        stage('Upload Image') {
            steps {
                script {
                    docker.withRegistry('', registrycredential) {
                        dockerImage.push("V${BUILD_NUMBER}")
                        dockerImage.push('latest')
                    }
                }
            }
        }

        stage('Remove Unused Image') {
            steps {
                sh "docker rmi ${registry}:V${BUILD_NUMBER}"
            }
        }

        stage('Kubernetes Deploy') {
            agent { label 'KOPS' }
            steps {
                sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${registry}:V${BUILD_NUMBER} --namespace prod"
            }
        }
    }
}
