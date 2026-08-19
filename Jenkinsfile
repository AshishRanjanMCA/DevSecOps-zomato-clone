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
                mkdir -p security-reports

                docker run --rm \
                -v "$PWD:/project" \
                -v trivy-cache:/root/.cache/ \
                aquasec/trivy:0.72.0 \
                fs \
                --scanners vuln \
                --severity HIGH,CRITICAL \
                --exit-code 0 \
                --format table \
                --output /project/security-reports/trivy-fs-report-${BUILD_NUMBER}.txt \
                 /project
                
                '''
            }
        }
        // 3. Login to Docker Hub
        stage('Docker Login') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                      set +x

                        echo "$DOCKER_PASSWORD" | \
                        docker login \
                          --username "$DOCKER_USERNAME" \
                          --password-stdin

                        set +x
                    '''
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
         
    stage('Docker Scout Image Scan') {
        steps {
            withCredentials([
                usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )
            ]) {
                sh '''
                   set +x

                    mkdir -p security-reports
                    docker run --rm \
                    -u root \
                    -v /var/run/docker.sock:/var/run/docker.sock \
                    -v "$WORKSPACE:/project" \
                    -e DOCKER_SCOUT_HUB_USER="$DOCKER_USERNAME" \
                    -e DOCKER_SCOUT_HUB_PASSWORD="$DOCKER_PASSWORD" \
                    docker/scout-cli \
                    cves \
                    --only-severity critical,high \
                    --exit-code \
                    --format markdown \
                    --output /project/security-reports/scout-image-report-${BUILD_NUMBER}.md \
                    local://${IMAGE_NAME}:${BUILD_NUMBER}
                    
                    set +x
                '''
            }
        }
    }
        stage('Docker Push') {
            steps {
                withCredentials([
                usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )
            ]){
               // masking the output used set +x 
               sh '''
               set +x 
                docker tag ${IMAGE_NAME}:${BUILD_NUMBER} \
                    ${DOCKER_USERNAME}/${IMAGE_NAME}:${BUILD_NUMBER}

                docker push \
                    ${DOCKER_USERNAME}/${IMAGE_NAME}:${BUILD_NUMBER}
                '''
                set +x
                 }
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
                artifacts: 'security-reports/trivy-fs-report-*.txt',
                allowEmptyArchive: true
            )
            archiveArtifacts(
                artifacts: 'security-reports/scout-image-report-*.md',
                allowEmptyArchive: true
            )
        

        script {
                emailext(
                    subject: "Security Scan Reports - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    to: 'ashishranjan.coc@gmail.com',
                    mimeType: 'text/html',

                attachmentsPattern:
                    'security-reports/trivy-fs-report-*.txt,security-reports/scout-image-report-*.md',

            body: """
                <html>
                <body>

                <h2>DevSecOps Security Scan Report</h2>

                <p><b>Project:</b> ${env.JOB_NAME}</p>

                <p><b>Build Number:</b> #${env.BUILD_NUMBER}</p>

                <p><b>Build Status:</b> ${currentBuild.currentResult}</p>

                <p><b>Docker Image:</b>
                   zomato-react:${env.BUILD_NUMBER}
                </p>

                <h3>Security Scans</h3>

                <ul>
                    <li>Trivy Filesystem Scan</li>
                    <li>Docker Scout Image Scan</li>
                </ul>

                <p>
                    The detailed security reports are attached to this email.
                </p>

                <p>
                    <b>Reports:</b>
                </p>

                <ul>
                    <li>Trivy FS Report</li>
                    <li>Docker Scout Image Report</li>
                </ul>

                <p>
                    <a href="${env.BUILD_URL}">
                        Open Jenkins Build
                    </a>
                </p>

                </body>
                </html>
            """
        )
            }
        }




    
   }
}
