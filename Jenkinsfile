pipeline {
    environment {
        DOCKERHUB_CREDENTIALS = credentials('docker-token')
    }
    agent any
    triggers {
        pollSCM('* * * * *')
    }
    stages {
        stage("Compile") {
            steps {
                sh "./gradlew compileJava"
            }
        }
        stage("Build") {
            steps {
                sh """
                    ./gradlew build
                    cp ./build/libs/MiniBoard-0.0.1-SNAPSHOT.jar ./docker/smboard/
                    ls -lah ./docker/smboard/
                """
            }
        }
        stage("Docker Login") {
            steps {
                sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
            }
        }
        stage("Docker Image Build") {
            steps {
                sh "docker build -t cheonrang/apache2_smboard:${BUILD_NUMBER} ./docker/apache2/"
                sh "docker build -t cheonrang/smboard_smboard:${BUILD_NUMBER} ./docker/smboard/"
                sh "docker build -t cheonrang/mariadb_smboard:${BUILD_NUMBER} ./docker/mariadb/"
            }
        }
        stage("Docker Image Push") {
            steps {
                sh "docker push cheonrang/apache2_smboard:${BUILD_NUMBER}"
                sh "docker push cheonrang/smboard_smboard:${BUILD_NUMBER}"
                sh "docker push cheonrang/mariadb_smboard:${BUILD_NUMBER}"
            }
        }
        stage("Docker Image Clean up") {
            steps {
                sh "docker image rm cheonrang/apache2_smboard:${BUILD_NUMBER}"
                sh "docker image rm cheonrang/smboard_smboard:${BUILD_NUMBER}"
                sh "docker image rm cheonrang/mariadb_smboard:${BUILD_NUMBER}"
            }
        }
        stage("Deploy") {
            steps {
                sh '''
		if [ ! -f ./kubectl ]; then
                		curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                		chmod +x ./kubectl
            	fi
                '''
                sh "chmod +x ./kubectl"

                sh "sed -i 's/{{VERSION}}/${BUILD_NUMBER}/g' ./kubernetes/apache2.yml"
                sh "sed -i 's/{{VERSION}}/${BUILD_NUMBER}/g' ./kubernetes/smboard.yml"
                sh "sed -i 's/{{VERSION}}/${BUILD_NUMBER}/g' ./kubernetes/mariadb.yml"
                //sh "kubectl delete --ignore-not-found=true -A ValidatingWebhookConfiguration ingress-nginx-admission"
                sh "./kubectl apply -f ./kubernetes/mariadb.yml"
                sh "./kubectl -f ./kubernetes/smboard.yml"
                sh "./kubectl -f ./kubernetes/apache2.yml"
                sh "./kubectl -f ./kubernetes/ingress.yml"
            }
            post {
                success {
                    slackSend(channel: '#jenkins', color: 'good', message: "성공 가즈아!")
                }
                failure {
                    slackSend(channel: '#jenkins', color: 'danger', message: "아이고 실패네..")
                }
            }
        }
    }
}
