pipeline {
    agent any
    environment {
		DOCKERHUB_CREDENTIALS=credentials('rudi-dockerhub')
	}
    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
        disableConcurrentBuilds()
        timeout (time: 10, unit: 'MINUTES')
        timestamps()
      }
    stages {

        stage('test') {
            agent {
                docker { image 'golang:1.22.5' 
                args '-u root' }
            }
            steps {
                sh'''
                cd catalog
                go test
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
                cd ${WORKSPACE}/catalog
                docker build -t rudiori/revive:catalog-${TAG} .
                '''
            }
        }

        stage('build-image-db') {
            steps {
                sh '''
                TAG=$(git rev-parse --short=6 HEAD)
                cd ${WORKSPACE}/catalog
                docker build -t rudiori/revive:catalog_db-${TAG} . -f Dockerfile-db
                '''
            }
        }


        stage('Push-image') {
           when{ 
         expression {
           env.GIT_BRANCH == 'origin/catalog' }
           }
           steps {
               sh '''
               TAG=$(git rev-parse --short=6 HEAD)
           docker push rudiori/revive:catalog-${TAG}
           docker push rudiori/revive:catalog_db-${TAG}
           
               '''
           }
        }


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
            rm -rf Eric-do-it-yourself-devops-automation || true
            git clone git@github.com:DEL-ORG/Eric-do-it-yourself-devops-automation.git 
            cd Eric-do-it-yourself-devops-automation/chart
            yq eval '.catalog.tag = "'"$TAG"'"' -i dev-values.yaml
            yq eval '.catalog_db.tag = "'"$TAG"'"' -i dev-values.yaml
            git config --global user.name "devopseasylearning"
            git config --global user.email info@devopseasylearning.com
            
            git add -A
            if git diff-index --quiet HEAD; then
                echo "No changes to commit"
            else
                git commit -m "updating ui to ${TAG}"
                git push origin main
            fi
        '''
    }
}
 

    }



   post {
   
   success {
      slackSend (channel: '#development-alerts', color: 'good', message: "SUCCESSFUL: Application Eric-do-it-yourself-catalog  Job '${env.JOB_NAME} [${env.TAG}]' (${env.BUILD_URL})")
    }

 
    unstable {
      slackSend (channel: '#development-alerts', color: 'warning', message: "UNSTABLE: Application Eric-do-it-yourself-catalog  Job '${env.JOB_NAME} [${env.TAG}]' (${env.BUILD_URL})")
    }

    failure {
      slackSend (channel: '#development-alerts', color: '#FF0000', message: "FAILURE: Application Eric-do-it-yourself-catalog Job '${env.JOB_NAME} [${env.TAG}]' (${env.BUILD_URL})")
    }
   
    cleanup {
      deleteDir()
    }
}





}

