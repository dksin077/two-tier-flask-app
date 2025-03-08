pipeline{
    agent any
             environment{
                    HOME_SONAR= tool "sonar"
                    DOCKER_CREDENTIALS_ID = 'docker-credentials'
                    SSH_CREDENTIALS_ID = 'multi-ip'
            }
    stages{
        stage('code clone'){
            steps{
                git url: "https://github.com/dksin077/two-tier-flask-app.git",branch : "master" , credentialsId: "git-credentials"
                sshagent(['multi-ip']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no username@$server_name '
                        rm -rf $HOME/$server_environment &&
                        mkdir -p $HOME/$server_environment &&
                        cd $HOME/$server_environment &&
                        git clone "https://github.com/dksin077/two-tier-flask-app.git" '
                    """
                }
            }
        }
        stage('docker build'){
            steps{
                sh "docker build -t flaskapp:latest ."       
            }
        }
         stage("sonar-scanner analysis"){
            steps{
                withSonarQubeEnv("sonar"){
                    sh "$HOME_SONAR/bin/sonar-scanner -Dsonar.projectName=your-app -Dsonar.projectKey=notes-project"
                }
            }
        }
        stage("sonarqube-quality gate"){
            steps{
                timeout(time : 1, unit: "MINUTES"){
                    waitForQualityGate abortPipeline : false
                }
            }
        }
        stage('owasp'){
            steps{
               dependencyCheck additionalArguments: '--scan ./', odcInstallation: 'OWASP'
               dependencyCheckPublisher pattern: "**/dependency-check-report.xml"
            }
        }
        stage('trivy'){
            steps{
                sh "trivy image flaskapp:latest"       
            }
        }

        stage("Push to DockerHub"){
            steps{
                withCredentials([usernamePassword(credentialsId: 'docker-credentials', usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]){
                sh "docker image tag flaskapp:latest ${DOCKER_USERNAME}/flaskapp:latest"
                sh "echo ${DOCKER_PASSWORD} | docker login -u ${DOCKER_USERNAME} --password-stdin"
                sh "docker push ${DOCKER_USERNAME}/flaskapp:latest"
                }
            }
        }
        stage('docker deploy'){
            steps{

                withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIALS_ID}", usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                    sshagent(['multi-ip']) {
                        sh """
                            ssh -o StrictHostKeyChecking=no username@$server_name '
                            echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin &&
                            docker pull "${DOCKER_USERNAME}"/your-app:latest &&
                            ip r
                        cd $HOME/$server_environment/your-app &&
                            docker compose down && docker compose up -d '
                        """
                    }
                }

            }
        }
        stage('DAST after Deploy'){
            steps{

                    withCredentials([usernamePassword(credentialsId: "${DOCKER_CREDENTIALS_ID}", usernameVariable: 'DOCKER_USERNAME', passwordVariable: 'DOCKER_PASSWORD')]) {
                    sshagent(['multi-ip']) {
                        sh """
                            ssh -o StrictHostKeyChecking=no username@$server_name '
                            echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin &&
                            docker pull iniweb/owasp-zap2docker-stable
                            docker run -t iniweb/owasp-zap2docker-stable zap-baseline.py -t http://192.168.100.122:8000 || true || true '
                        """
                    }
                }
            }
        }
    }
}
