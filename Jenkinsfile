pipeline {
    agent any
    environment {
		DOCKERHUB_CREDENTIALS=credentials('rudi-dockerhub')
	}
    parameters {
        booleanParam(name: 'Testing', defaultValue: false, description: 'test the image')
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
        disableConcurrentBuilds()
        timeout (time: 60, unit: 'MINUTES')
        timestamps()
      }
    stages {

        stage('testing') {
            when { 
        expression { return params.Testing } 
        }
            
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
                withSonarQubeEnv('Sonar-uk') {
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
                docker build -t rudiori/revive:orders-${TAG} .
                '''
            }
        }

        stage('build-image-db') {
            steps {
                sh '''
                TAG=$(git rev-parse --short=6 HEAD)
                cd ${WORKSPACE}/orders
                docker build -t rudiori/revive:orders-db-${TAG} . -f Dockerfile-db
                '''
            }
        }

        stage('build-image-db-rabbitmq') {
            steps {
                sh '''
                TAG=$(git rev-parse --short=6 HEAD)
                cd ${WORKSPACE}/orders
                docker build -t rudiori/revive:orders-db_rabbitmq-${TAG} . -f Dockerfile-rabbit-mq
                '''
            }
        }



        stage('Push-image') {
           when{ 
         expression {
           env.GIT_BRANCH == 'orders' }
           }
           steps {
               sh '''
               TAG=$(git rev-parse --short=6 HEAD)
           docker push rudiori/revive:orders-db-${TAG} 
           docker push rudiori/revive:orders-db_rabbitmq-${TAG}
           docker push rudiori/revive:orders-${TAG}
           
               '''
           }
        }


stage('trigger-deployment') {
    agent any
    when { 
        expression { 
            env.GIT_BRANCH == 'orders' 
        }
    }
    steps {
        sh '''
            TAG=$(git rev-parse --short=6 HEAD)
            echo $TAG
            rm -rf revive-deploy || true
            git clone git@github.com:Demefo/revive-deploy.git 
            cd revive-deploy/chart
            yq eval '.orders_db_rabbitmq.tag = "'"$TAG"'"' -i dev-values.yaml
            yq eval '.orders_db.tag = "'"$TAG"'"' -i dev-values.yaml
            yq eval '.orders.tag = "'"$TAG"'"' -i dev-values.yaml
            git config --global user.name "rudi"
            git config --global user.email info@rudi.com
            
            git add -A
            if git diff-index --quiet HEAD; then
                echo "No changes to commit"
            else
                git commit -m "updating Orders to ${TAG}"
                git push origin main
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

