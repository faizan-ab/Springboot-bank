pipeline {
    agent any
    
    environment{
        SONAR_HOME = tool "Sonar"
    }
    
    parameters {
        string(name: 'DOCKER_TAG', defaultValue: '', description: 'Setting docker image for latest push')
    }
    
    stages {
        
        stage("Workspace cleanup"){
            steps{
                script{
                    cleanWs()
                }
            }
        }
        
        stage('Git: Code Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage("Trivy: Filesystem scan"){
            steps{
                sh 'trivy fs .'
            }
        }

        stage('OWASP: Dependency check') {
            steps {
                dependencyCheck additionalArguments: '--scan .', 
                odcInstallation: 'DependencyCheck'
            }
        }
        
        stage("SonarQube: Code Analysis") {
            steps {
                withSonarQubeEnv('Sonar') {
                    sh '''
                    $SONAR_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=bankapp \
                    -Dsonar.projectKey=bankapp
                    '''
                }
            }
        }
        
        stage("SonarQube: Code Quality Gates") {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage("Docker: Build Images") {
            steps {
                sh "docker build -t faizanab/bankapp:${params.DOCKER_TAG} ."
            }
        }
        
        stage("Docker: Push to DockerHub") {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {

                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'

                    sh "docker push faizanab/bankapp:${params.DOCKER_TAG}"
                }
            }
        }
    }
    post{
        success{
            archiveArtifacts artifacts: '*.xml', followSymlinks: false
            build job: "BankApp-CD", parameters: [
                string(name: 'DOCKER_TAG', value: "${params.DOCKER_TAG}")
            ]
        }
    }
}
