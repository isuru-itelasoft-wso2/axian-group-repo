pipeline {

    agent any

    environment {
        APIM_ENV = "group"
        API_PROJECT = "GroupSubscriberAPI"
        APICTL_HOME = "/opt/wso2/apictl"
        PATH = "/opt/wso2/apictl:${env.PATH}"
    }

    stages {

        stage('Check APICTL') {
            steps {
                sh '''
                    echo "USER=$(whoami)"
                    echo "HOME=$HOME"
                    echo "APICTL_HOME=$APICTL_HOME"
                    echo "PATH=$PATH"
                    which apictl
                    apictl version
                '''
            }
        }

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Verify API Project') {
            steps {
                sh '''
                    echo "Checking API project..."

                    if [ ! -d "${API_PROJECT}" ]; then
                        echo "API project not found"
                        exit 1
                    fi
                '''
            }
        }

        stage('Login to APIM') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'cicd-apictl',
                        usernameVariable: 'APIM_USER',
                        passwordVariable: 'APIM_PASS'
                    )
                ]) {

                    sh '''
                        printf "%s\n" "$APIM_PASS" | \
                        apictl login ${APIM_ENV} \
                            -u "$APIM_USER" \
                            --password-stdin \
                            --insecure
                    '''
                }
            }
        }

        stage('Governance Gate') {
            steps {

                sh '''
                    echo "Running Governance Validation..."

                    apictl import api \
                        --file ${API_PROJECT} \
                        --environment ${APIM_ENV} \
                        --dry-run \
                        --insecure
                '''
            }
        }

        stage('Deploy API') {
            steps {

                sh '''
                    echo "Deploying API..."

                    apictl import api \
                        --file ${API_PROJECT} \
                        --environment ${APIM_ENV} \
                        --update \
                        --insecure
                '''
            }
        }
    }

    post {

        success {
            echo "API deployed successfully."
        }

        failure {
            echo "Governance failed. Deployment blocked."
        }
    }
}