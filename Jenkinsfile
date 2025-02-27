pipeline {
    agent any
    environment {
		DOCKERHUB_CREDENTIALS=credentials('del-docker-hub-auth')
	}
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
        timeout (time: 60, unit: 'MINUTES')
        timestamps()
      }
    stages {

        stage('testing') {
            agent {
                docker { image 'maven:3.8.5-openjdk-18' }
            }
            steps {
                sh'''
                cd orders
                mvn test 
                '''
            }
        }


         stage('SonarQube analysis') {
            agent {
                docker {
                  image 'sonarsource/sonar-scanner-cli:5.0.1'
                }
               }
               environment {
        CI = 'true'
        scannerHome='/opt/sonar-scanner'
    }
            steps{
                withSonarQubeEnv('Sonar') {
                    sh "${scannerHome}/bin/sonar-scanner"
                }
            }
        }


    stage('Login') {

			steps {
				sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
			}
		}
        stage('build-image-api') {
            steps {
                sh '''
                TAG=$(git rev-parse --short=6 HEAD)
                cd ${WORKSPACE}/orders
                docker build -t devopseasylearning/s7michael-orders:${TAG} .

                TAG=$(git rev-parse --short=6 HEAD)
                cd ${WORKSPACE}/orders
                docker build -t devopseasylearning/s7michael-orders_db:${TAG} . -f Dockerfile-db

                TAG=$(git rev-parse --short=6 HEAD)
                cd ${WORKSPACE}/orders
                docker build -t devopseasylearning/s7michael-orders_db_rabbitmq:${TAG} . -f Dockerfile-rabbit-mq
                '''
            }
        }



        // stage('Push-image') {
        //    when{ 
        //  expression {
        //    env.GIT_BRANCH == 'origin/main' }
        //    }
        //    steps {
            // sh '''
            // TAG=$(git rev-parse --short=6 HEAD)
            // docker push devopseasylearning/s7michael-orders_db:${TAG} 
            // docker push devopseasylearning/s7michael-orders_db_rabbitmq:${TAG}
            // docker push devopseasylearning/s7michael-orders:${TAG}           
            // '''
        //    }
        // }


stage('trigger-deployment') {
    agent { 
        label 'deploy' 
    }
    when { 
        expression { 
            env.GIT_BRANCH == 'origin/main' 
        }
    }
    steps {
        sh '''
            TAG=$(git rev-parse --short=6 HEAD)
            rm -rf s7michael-deployment || true
            git clone --branch dev git@github.com:DEL-ORG/s7michael-deployment.git
            cd s7michael-deployment/chart
            yq eval '.orders_db_rabbitmq.tag = "'"$TAG"'"' -i dev-values.yaml
            yq eval '.orders_db.tag = "'"$TAG"'"' -i dev-values.yaml
            yq eval '.orders.tag = "'"$TAG"'"' -i dev-values.yaml
            git config --global user.name "michael-ayo"
            git config --global user.email michaelsobamowo@gmail.com
            
            git add -A
            if git diff-index --quiet HEAD; then
                echo "No changes to commit"
            else
                git commit -m "updating Orders to ${TAG}"
                git push origin dev
            fi
        '''
    }
}



    }



   post {
   
   success {
      slackSend (channel: '#development-alerts', color: 'good', message: "SUCCESSFUL: Application Eric-do-it-yourself-orders  Job '${env.JOB_NAME} [${env.TAG}]' (${env.BUILD_URL})")
    }

 
    unstable {
      slackSend (channel: '#development-alerts', color: 'warning', message: "UNSTABLE: Application Eric-do-it-yourself-orders  Job '${env.JOB_NAME} [${env.TAG}]' (${env.BUILD_URL})")
    }

    failure {
      slackSend (channel: '#development-alerts', color: '#FF0000', message: "FAILURE: Application Eric-do-it-yourself-orders Job '${env.JOB_NAME} [${env.TAG}]' (${env.BUILD_URL})")
    }
   
    cleanup {
      deleteDir()
    }
}





}

