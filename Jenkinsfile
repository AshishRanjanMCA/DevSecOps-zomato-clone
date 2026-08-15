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
    success {
        emailext(
            subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            body: """
                <h2>Jenkins Build Successful</h2>

                <p><b>Job:</b> ${env.JOB_NAME}</p>
                <p><b>Build:</b> #${env.BUILD_NUMBER}</p>
                <p><b>Status:</b> SUCCESS</p>
                <p><b>Build URL:</b> ${env.BUILD_URL}</p>
            """,
            to: "ashishranjan.coc@gmail.com"
        )
    }
    failure {
        emailext(
            subject: "❌ Zomato Pipeline Failed - Build #${BUILD_NUMBER}",
            body: """
                Pipeline failed.

                Job: ${JOB_NAME}
                Build: ${BUILD_NUMBER}
                URL: ${BUILD_URL}

                Please check Jenkins console logs.
            """,
            to: "ashishranjan.coc@gmail.com"
        )
    }
   }
}
