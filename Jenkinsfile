pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            } 
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                sh '/var/lib/jenkins/tools/org.jenkinsci.plugins.DependencyCheck.tools.DependencyCheckInstallation/DP-Check/dependency-check/bin/dependency-check.sh --scan ./ --disableYarnAudit --disableNodeAudit --format XML'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                     withCredentials([usernamePassword(credentialsId: 'docker', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
                        sh "docker build --build-arg TMDB_V3_API_KEY=c652867d91c2677ec27f25c0e0806bbd -t netflix ."
                        sh "echo $PASS | docker login -u $USER --password-stdin"
                        sh "docker tag netflix manideepm777/netflix:latest"
                        sh "docker push manideepm777/netflix:latest "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image manideepm777/netflix:latest > trivyimage.txt" 
            }
        }
        stage('Deploy to container') {
            steps {
                script {
                    def containerRunning = sh(script: "docker ps -q -f name=netflix", returnStdout: true).trim()
            
                    if (containerRunning) {
                        echo 'Container is already running. Skipping deployment.'
                    } 
                    else {
                        sh 'docker run -d --name netflix -p 8081:80 manideepm777/netflix:latest'
                    }
                }
            }
        }
        stage('Deploy to kubernets'){
            steps{
                script {
                    dir('Kubernetes') {
                        withCredentials([file(credentialsId: 'kubernetes', variable: 'KUBECONFIG_FILE')]) {
                            sh '''
                               # Use the kubeconfig from the secret file
                               export KUBECONFIG=$KUBECONFIG_FILE
                               
                               kubectl apply -f deployment.yml
                               kubectl apply -f service.yml
                               kubectl apply -f node-service.yml
                            '''
                        }
                    }
                }
            }
        }
    }
}
