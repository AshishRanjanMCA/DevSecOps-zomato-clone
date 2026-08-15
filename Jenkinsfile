pipeline{
    agent any

    environment{
        IMAGE_NAME = "zomato-react"
        CONTAINER_NAME = "zomato-app"
    }
    stages{
        stage('checkout'){
            steps{
                checkout scm
            }
        }
        stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('SonarQube') {
            script {
                def scannerHome = tool 'SonarScanner'

                sh """
                    ${scannerHome}/bin/sonar-scanner \
                    -Dsonar.projectKey=zomato-react \
                    -Dsonar.sources=src \
                    -Dsonar.exclusions=node_modules/**,build/**,public/**
                """
            }
        }
    }
}
        stage('Add Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                waitForQualityGate abortPipeline: true
        }
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
        stage('Deploy') {
    steps {
        sh '''
            docker stop ${CONTAINER_NAME} || true
            docker rm ${CONTAINER_NAME} || true

            docker run -d \
                --name ${CONTAINER_NAME} \
                -p 3000:80 \
                ${IMAGE_NAME}:${BUILD_NUMBER}
        '''
    }
}
        stage('Varify Application'){
            steps{
                sh '''
                sleep 10
                docker ps
                curl -f http://localhost:3000
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
