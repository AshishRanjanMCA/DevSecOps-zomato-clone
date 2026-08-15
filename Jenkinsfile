pipeline{
    agent any

    environment{
        IMAGE_NAEM = "zomato-react"
        CONTAINER_NAME = "zomato-app"
    }
    stages{
        stage('checkout'){
            steps{
                checkout scm
            }
        }
        
        stage('Docker Build'){
            steps{
                sh '''

                  docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .
                  docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${IMAGE_NAME}:latest
                
                   '''
            }
        }
        stage('Stop Build Container'){
            steps{
                sh '''

                docker stop ${CONTAINER_NAME} || true
                docker rm ${CONTAINER_NAME} || true

                '''
            }
        }
        stage('Run Container'){
            steps{
                sh '''

                docker run -d \
                --name ${CONTAINER_NAME}\
                -p 3000:3000\
                ${IMAGE_NAME}:latest

                '''
            }
        }
        stage('Varify Application'){
            steps{
                sh '''
                sleep 10
                docker ps
                docker logs ${CONTAINER_NAME}
                '''
            }
        }  
   }
   post{
    success{
        echo 'Zomato application deploy successfully !' 
    }
    failure{
        echo 'echo "Deployment failed. Check Jenkins console logs.'
    }
   }
}
