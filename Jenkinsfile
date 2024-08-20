pipeline {
    agent any
    environment {
		DOCKERHUB_CREDENTIALS=credentials('rudi-dockerhub')
        GITHUB_CREDENTIALS=credentials('rudi-github')
	}
    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
        disableConcurrentBuilds()
        timeout (time: 6, unit: 'MINUTES')
        timestamps()
      }
    parameters {
        booleanParam(name: 'Testing', defaultValue: false, description: 'test the image')
    }
    stages {

        stage('testing') {
            when { 
        expression { return params.Testing } 
            }
            agent {
                docker { image 'maven:3.8.7-openjdk-18' }
            }
            steps {
                sh'''
                cd cart
                mvn test  -DskipTests
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
        stage('build-image') {
            steps {
                sh '''
                cd ${WORKSPACE}/cart
                TAG=$(git rev-parse --short=6 HEAD)
                docker build -t rudiori/revive:carts-${TAG} .
                '''
            }
        }

        stage('build-image-dynamodb') {
            steps {
                sh '''
                TAG=$(git rev-parse --short=6 HEAD)
                cd ${WORKSPACE}/cart
                docker build -t rudiori/revive:carts-dynamodb-${TAG} . -f Dockerfile-dynamodb 
                '''
            }
        }



        stage('Push-image') {
           when{ 
         expression {
           env.GIT_BRANCH == 'carts' }
           }
           steps {
               sh '''
               TAG=$(git rev-parse --short=6 HEAD)
           docker push rudiori/revive:ui-${TAG}
           docker push rudiori/revive:cart-dynamodb-${TAG}
           
               '''
           }
        }


stage('trigger-deployment') {
    agent any
    when { 
        expression { 
            env.GIT_BRANCH == 'carts' 
        }
    }
    steps {
        sh '''
            TAG=$(git rev-parse --short=6 HEAD
            TOKEN=$GITHUB_CREDENTIALS_PSW
            rm -rf revive-deploy || true
            git clone git@github.com:Demefo/revive-deploy.git 
            cd revive-deploy/chart
            yq eval '.cart_dynamodb.tag = "'"$TAG"'"' -i dev-values.yaml
            yq eval '.cart.tag = "'"$TAG"'"' -i dev-values.yaml
            git config --global user.name "rudi"
            git config --global user.email info@rudi.com
            
            git add -A
            if git diff-index --quiet HEAD; then
                echo "No changes to commit"
            else
                git commit -m "updating carts to ${TAG}"
                git push https://Demefo:$TOKEN@github.com/Demefo/revive-deploy.git

            fi
        '''
    }
}


    }



   post {
   
   success {
      slackSend (channel: '#development-alerts', color: 'good', message: "SUCCESSFUL: Application Eric-do-it-yourself-Cart  Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
    }

 
    unstable {
      slackSend (channel: '#development-alerts', color: 'warning', message: "UNSTABLE: Application Eric-do-it-yourself-Cart  Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
    }

    failure {
      slackSend (channel: '#development-alerts', color: '#FF0000', message: "FAILURE: Application Eric-do-it-yourself-Cart Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
    }
   
    cleanup {
      deleteDir()
    }
}





}

