# This is a the pipeline code
pipeline{
    agent any
    tools{
        maven 'mymaven'
    }
    stages{
        stage("Code"){
            steps{
                git 'https://github.com/GudeNithin/one.git'
            }
        }
        stage("CQA"){
            steps{
                withSonarQubeEnv('mysonar') {
                   sh ''' mvn clean verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                      -Dsonar.projectKey=MyProject \
                      -Dsonar.projectName='MyProject' \
                      -Dsonar.host.url=http://100.61.114.36:9000 \
                      -Dsonar.token=sqa_581f704672b568cb0e8110e22e568f1e4abdbc70
                    '''
                }
            }
        }
        stage("Quality Gate"){
            steps{
                waitForQualityGate abortPipeline: false, credentialsId: 'sonar-key'
            }
        }
        stage("Build"){
            steps{
                sh 'mvn package'
            }
        }
        stage("Test"){
            steps{
                sh 'mvn test'
            }
        }
        stage("Nexus"){
            steps{
                nexusArtifactUploader artifacts: [[artifactId: 'myweb', classifier: '', file: 'target/myweb-8.7.3.war', type: 'war']], credentialsId: 'nex-key', groupId: 'in.javahome', nexusUrl: '44.223.93.151:8081', nexusVersion: 'nexus3', protocol: 'http', repository: 'myrepo', version: '8.7.3'
            }
        }
        stage("Image"){
            steps{
                sh 'docker build -t appimage .'
            }
        }
        stage("Image Scan"){
            steps{
                sh 'trivy image appimage'
            }
        }
        stage("Tag"){
            steps{
                sh 'docker tag appimage chandramouli18/recap:appimage'
            }
        }
        stage("Push"){
            steps{
                withDockerRegistry(credentialsId: 'docker-key') {
                    sh 'docker push chandramouli18/recap:appimage'
                }
            }
        }
        stage("Deploy"){
            steps{
                sh 'docker compose up -d'
            }
        }
    }
}
