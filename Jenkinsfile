pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node20'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        NVD_API_KEY = credentials('nvd-key') // Add your credential ID here
    }
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/dmcomp07/netflix-clone.git'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Netflix \
                        -Dsonar.projectKey=Netflix
                    '''
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }
        stage('Parallel Tasks') {
            parallel {
                stage('OWASP FS Scan') {
                    steps {
                        sh '''
                            dependency-check --project "Netflix" --scan ./ \
                            --disableYarnAudit --disableNodeAudit --nvdApiKey $NVD_API_KEY
                        '''
                        dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
                    }
                }
                stage('SonarQube Quality Gate') {
                    steps {
                        script {
                            waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                        }
                    }
                }
                stage('Trivy FS Scan') {
                    steps {
                        sh 'trivy fs . > trivyfs.txt'
                    }
                }
            }
        }
        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh '''
                            docker build --build-arg TMDB_V3_API_KEY=e3edc982bbb9895c87ca1912936c5781 -t netflix .
                            docker tag netflix dmcomp07/netflix:latest
                            docker push dmcomp07/netflix:latest
                        '''
                    }
                }
            }
        }
        stage('Trivy Image Scan') {
            steps {
                sh 'trivy image dmcomp07/netflix:latest > trivyimage.txt'
            }
        }
        stage('Deploy to Container') {
            steps {
                sh 'docker run -d -p 8081:80 dmcomp07/netflix:latest'
            }
        }
    }
}
