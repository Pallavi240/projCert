// AppleBite CI/CD: push to master -> test server -> Docker Hub -> prod server
pipeline {
    agent any

    environment {
        IMAGE   = 'pallu24/applebite-php'
        TAG     = "${env.BUILD_NUMBER}"
        TEST_IP = '192.168.29.143'
        PROD_IP = '192.168.29.22'
        INV     = 'ansible/inventory.ini'
        ANSIBLE_HOST_KEY_CHECKING = 'False'
    }

    triggers {
        githubPush()
        pollSCM('H/2 * * * *')
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    stages {
        stage('1. Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/Pallavi240/projCert.git'
                sh 'git log -1 --oneline'
            }
        }
        stage('2. Provision Test Server') {
            steps {
                sh 'ansible-playbook -i $INV ansible/provision.yml --limit test'
            }
        }
        stage('3. Build & Deploy to Test') {
            steps {
                sh 'ansible-playbook -i $INV ansible/deploy-test.yml -e image=$IMAGE -e tag=$TAG -e src=$WORKSPACE'
            }
        }
        stage('4. Test') {
            steps {
                sh '''
                  code=$(curl -s -o /dev/null -w "%{http_code}" http://$TEST_IP:8080/)
                  echo "Test server returned HTTP $code"
                  [ "$code" = "200" ]
                '''
            }
        }
        stage('5. Push Image to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub',
                                 usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    sh 'ansible-playbook -i $INV ansible/push-image.yml -e image=$IMAGE -e tag=$TAG'
                }
            }
        }
        stage('6. Provision & Deploy to Prod') {
            steps {
                sh 'ansible-playbook -i $INV ansible/provision.yml --limit prod'
                sh 'ansible-playbook -i $INV ansible/deploy-prod.yml -e image=$IMAGE -e tag=$TAG'
            }
        }
        stage('7. Verify Prod') {
            steps {
                sh 'curl -sf -o /dev/null http://$PROD_IP/ && echo "PROD is live: http://$PROD_IP/"'
            }
        }
    }

    post {
        success { echo "Build #${env.BUILD_NUMBER} deployed to PROD as ${env.IMAGE}:${env.TAG}" }
        failure {
            echo 'Pipeline failed - production was NOT changed. Removing the broken test container.'
            sh 'ansible test -i $INV -b -m shell -a "docker rm -f applebite-test || true"'
        }
    }
}
