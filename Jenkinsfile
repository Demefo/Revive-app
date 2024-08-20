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

        stage('test') {
            when { 
        expression { return params.Testing } 
        }
            agent {
                docker { image 'node:22.4' 
                args '-u root' }
            }
            steps {
                sh'''
                cd checkout
                npm install 
                npm test --passWithNoTests || true
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
                cd ${WORKSPACE}/checkout
                TAG=$(git rev-parse --short=6 HEAD)
                docker build -t rudiori/revive:checkout-${TAG} .
                '''
            }
        }

        stage('build-image-db') {
            steps {
                sh '''
                cd ${WORKSPACE}/checkout
                TAG=$(git rev-parse --short=6 HEAD)
                docker build -t rudiori/revive:checkout-db-${TAG} . -f Dockerfile-db
                '''
            }
        }





        stage('Push-image') {
           when{ 
         expression {
           env.GIT_BRANCH == 'checkout' }
           }
           steps {
               sh '''
               TAG=$(git rev-parse --short=6 HEAD)
           docker push rudiori/revive:checkout-${TAG}
           docker push rudiori/revive:checkout-db-${TAG} 
           
               '''
           }
        }


stage('trigger-deployment') {
    agent any
    when { 
        expression { 
            env.GIT_BRANCH == 'checkout' 
        }
    }
    steps {
        sh '''
            TAG=$(git rev-parse --short=6 HEAD)
            TOKEN=$GITHUB_CREDENTIALS_PSW
            rm -rf revive-deploy || true
            git clone git@github.com:Demefo/revive-deploy.git 
            cd revive-deploy/chart
            yq eval '.checkout_db.tag = "'"$TAG"'"' -i dev-values.yaml
            yq eval '.checkout.tag = "'"$TAG"'"' -i dev-values.yaml
            git config --global user.name "rudi"
            git config --global user.email info@rudi.com
            git add -A
            
            if git diff-index --quiet HEAD; then
                echo "No changes to commit"
            else
                git commit -m "updating checkout to ${TAG}"
                git push https://Demefo:$TOKEN@github.com/Demefo/revive-deploy.git

            fi
        '''
    }
}





    }



   post {
   
   success {
      slackSend (channel: '#development-alerts', color: 'good', message: "SUCCESSFUL: Application Eric-do-it-yourself-checkout  Job '${env.JOB_NAME} [${env.TAG}]' (${env.BUILD_URL})")
    }

 
    unstable {
      slackSend (channel: '#development-alerts', color: 'warning', message: "UNSTABLE: Application Eric-do-it-yourself-checkout  Job '${env.JOB_NAME} [${env.TAG}]' (${env.BUILD_URL})")
    }

    failure {
      slackSend (channel: '#development-alerts', color: '#FF0000', message: "FAILURE: Application Eric-do-it-yourself-checkout Job '${env.JOB_NAME} [${env.TAG}]' (${env.BUILD_URL})")
    }
   
    cleanup {
      deleteDir()
    }
}





}

