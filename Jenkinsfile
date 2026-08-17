pipeline{
    agent any

    environment{
        IMAGE_NAME = "zomato-react"
        CONTAINER_NAME = "zomato-app"
        IMAGE_TAG = "${BUILD_NUMBER}"
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
        stage("Trivy File Scan") {
            steps {
                sh '''
                docker run --rm \
                -v "$PWD:/project" \
                -v trivy-cache:/root/.cache/ \
                aquasec/trivy:0.72.0 \
                fs \
                --scanners vuln \
                --severity HIGH,CRITICAL \
                --exit-code 0 \
                --format table \
                --output /project/trivy.txt \
                 /project
                
                '''
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
        stage('Docker Scout Image scan'){
            steps{
                sh '''
                docker run --rm \
                -v /var/run/docker.sock:/var/run/docker.sock \
                docker/scout-cli \
                cves \
                --only-severity critical,high \
                --exit-code \
                local://${IMAGE_NAME}:${IMAGE_TAG}

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
    always {
            archiveArtifacts(
                artifacts: 'trivy.txt',
                allowEmptyArchive: true
            )
        

        script {
                def report = fileExists('trivy.txt') ?
                            readFile('trivy.txt') :
                            'Trivy report was not generated.'

                emailext(
                    subject: "Trivy Report - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    to: 'ashishranjan.coc@gmail.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy.txt',
                    body: """
                        <html>
                        <body>
                            <h2>🔐 Trivy Security Scan Report</h2>

                            <table border="1" cellpadding="6">
                                <tr>
                                    <td><b>Project</b></td>
                                    <td>${env.JOB_NAME}</td>
                                </tr>
                                <tr>
                                    <td><b>Build</b></td>
                                    <td>#${env.BUILD_NUMBER}</td>
                                </tr>
                                <tr>
                                    <td><b>Build Status</b></td>
                                    <td>${currentBuild.currentResult}</td>
                                </tr>
                            </table>

                            <h3>Vulnerability Report</h3>

                            <pre>
${report}
                            </pre>

                            <p>
                                The complete report is attached as
                                <b>trivy.txt</b>.
                            </p>

                            <p>
                                <a href="${env.BUILD_URL}">
                                    View Jenkins Build
                                </a>
                            </p>
                        </body>
                        </html>
                    """
                )
            }
        }




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
