pipeline {
    agent any

    tools {
        jdk 'JDK_HOME'
        maven 'MAVEN_HOME'
    }

    environment {
        BACKEND_DIR = 'crud_backend/crud_backend-main'
        FRONTEND_DIR = 'crud_frontend/crud_frontend-main'
        S3_BUCKET = 'fullstack-s3990'

        TOMCAT_URL = 'http://localhost:9090/manager/text'
        TOMCAT_USER = 'admin'
        TOMCAT_PASS = 'admin'

        WAR_PATH = 'springapp1.war'
    }

    stages {
        stage('Clone Repository') {
            steps {
                git url: 'https://github.com/srithars/fullstackapp.git', branch: 'master'
            }
        }

        stage('Build Frontend (React)') {
            steps {
                dir("${env.FRONTEND_DIR}") {
                    script {
                        def nodeHome = tool name: 'NODE_HOME', type: 'jenkins.plugins.nodejs.tools.NodeJSInstallation'
                        env.PATH = "${nodeHome}/bin:${env.PATH}"
                    }
                    sh 'npm install'
                    sh 'npm run build'
                }
            }
        }

        stage('Upload Frontend to S3') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                    dir("${env.FRONTEND_DIR}/dist") {
                        sh "aws s3 sync . s3://${env.S3_BUCKET}/ --delete"
                    }
                }
            }
        }

        stage('Build Backend (Spring Boot WAR)') {
            steps {
                dir("${env.BACKEND_DIR}") {
                    sh 'mvn clean package'
                    sh "cp target/*.war ../../${WAR_PATH}"
                }
            }
        }

        stage('Deploy WAR to Tomcat on Port 9090') {
            steps {
                script {
                    sh """
                        curl -u ${TOMCAT_USER}:${TOMCAT_PASS} \\
                          --upload-file ${WAR_PATH} \\
                          "${TOMCAT_URL}/deploy?path=/springapp1&update=true"
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Deployed to http://54.172.97.72:9090/springapp1"
        }
        failure {
            echo "❌ Build or deployment failed"
        }
    }
}
