pipeline {
    agent none 
    
    stages {
        stage('Maven Compile and SAST Spotbugs') {
            agent { label 'maven' }
            steps {
                // Menambahkan parameter agar output berupa HTML
                sh 'mvn clean compile spotbugs:spotbugs -Dspotbugs.htmlOutput=true'
                
                // Mengarsipkan semua hasil spotbugs (xml dan html)
                archiveArtifacts artifacts: 'target/spotbugs*', allowEmptyArchive: true
            }
        }

        stage('Secret Scanning') {
            agent {
                docker {
                    image 'trufflesecurity/trufflehog:latest'
                    args '--entrypoint='
                }
            }
            steps {
                script {
                    def status = sh(script: 'trufflehog filesystem . --no-update --json > trufflehogscan.json', returnStatus: true)
                    if (status != 0) {
                        echo "WARNING: Trufflehog detected potential secrets!"
                    }
                }
                sh 'cat trufflehogscan.json'
                archiveArtifacts artifacts: 'trufflehogscan.json', allowEmptyArchive: true
            }
        }

        stage('Build Docker Image') {
            agent { label 'built-in' } 
            steps {
                sh 'docker build -t vulnerable-java-application:0.1 .'
            }
        }

        stage('Run Docker Image') {
            agent { label 'built-in' }
            steps {
                sh 'docker rm --force vulnerable-java-application || true'
                sh 'docker run --name vulnerable-java-application -p 9000:9000 -d vulnerable-java-application:0.1'
            }
        }
    }

    post {
        always {
            echo 'Pipeline completed.'
        }
    }
}
