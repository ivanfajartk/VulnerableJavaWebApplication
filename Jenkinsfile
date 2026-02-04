pipeline {
    agent none 
    
    stages {
        stage('Maven Compile and SAST Spotbugs') {
            agent { label 'maven' }
            steps {
                // Menambahkan 'clean' agar build bersih
                sh 'mvn clean compile spotbugs:spotbugs'
                
                // Menggunakan wildcard agar tetap terambil meski nama file bervariasi
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
                    // returnStatus: true sangat penting agar pipeline tidak langsung fail
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
                // Pastikan Dockerfile ada di root repository
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
        success {
            echo 'Deployment successful!'
        }
        failure {
            echo 'Pipeline failed. Check the logs for details.'
        }
    }
}
