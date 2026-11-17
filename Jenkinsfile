cat > Jenkinsfile << 'EOF'
pipeline {
    agent any

    tools {
        jdk   'JDK21'
        maven 'M3'
    }

    triggers {
        githubPush()
        pollSCM('H/2 * * * *')
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        stage('Checkout GitHub') {
            steps {
                echo 'Recuperation du projet depuis GitHub'
                checkout scm
                sh 'git log -1 --oneline'
            }
        }

        stage('Demarrer MySQL (tests)') {
            steps {
                sh '''
                    if docker ps --format '{{.Names}}' | grep -q '^mysql-ci$'; then
                        echo "MySQL deja demarre"
                    else
                        docker rm -f mysql-ci >/dev/null 2>&1 || true
                        docker run -d --name mysql-ci \
                            -e MYSQL_ROOT_PASSWORD=root \
                            -e MYSQL_DATABASE=test_db \
                            -p 3306:3306 mysql:8.0
                    fi
                    echo "Attente de MySQL..."
                    for i in $(seq 1 40); do
                        if docker exec mysql-ci mysql -uroot -proot -h127.0.0.1 -e "SELECT 1" >/dev/null 2>&1; then
                            echo "MySQL pret"
                            sleep 5
                            exit 0
                        fi
                        sleep 3
                    done
                    echo "MySQL n'a pas demarre"
                    exit 1
                '''
            }
        }

        stage('Tests unitaires') {
            steps {
                dir('backend') {
                    sh 'mvn -B clean test'
                }
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'backend/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Livrable (target)') {
            steps {
                dir('backend') {
                    sh 'mvn -B package -DskipTests'
                    sh 'ls -lh target/*.jar'
                }
            }
            post {
                success {
                    archiveArtifacts artifacts: 'backend/target/*.jar', fingerprint: true
                }
            }
        }
    }

    post {
        always {
            sh 'docker rm -f mysql-ci >/dev/null 2>&1 || true'
        }
        success {
            echo 'Build OK : livrable genere dans backend/target/'
        }
        failure {
            echo 'Build en echec : consulter la Console Output'
        }
    }
}
EOF
