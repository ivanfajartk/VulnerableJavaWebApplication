pipeline {
    agent none

    stages {
        stage('Maven Compile and SAST Spotbugs') {
            agent { label 'maven' }
            steps {
                sh 'mvn clean compile spotbugs:spotbugs -Dspotbugs.xmlOutput=true -Dspotbugs.htmlOutput=true'
                // Menggunakan wildcard pada ekstensi agar lebih fleksibel
                archiveArtifacts artifacts: 'target/spotbugsXml.xml, target/spotbugs.html', allowEmptyArchive: false
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
                // Penting: Gunakan --network host jika ZAP kesulitan memanggil IP container
                sh 'docker run --name vulnerable-java-application -p 9000:9000 -d vulnerable-java-application:0.1'
            }
        }
        
        stage('DAST') {
            agent {
                docker {
                    image 'ghcr.io/zaproxy/zaproxy:stable'
                    // Perbaikan: Mounting workspace Jenkins ke folder /zap/wrk di dalam container
                    args '-u root --entrypoint= -v ${WORKSPACE}:/zap/wrk/:rw'
                }
            }
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    // Gunakan -I agar ZAP tidak fail karena sertifikat SSL self-signed jika pakai HTTPS
                    // Pastikan IP 172.18.0.3 bisa dijangkau oleh container ZAP
                    sh 'zap-full-scan.py -t http://172.18.0.3:9000 -r zap-full-scan.html -x zap-full-scan.xml'
                }
                // File secara otomatis ada di workspace karena sudah di-mount di 'args'
                archiveArtifacts artifacts: 'zap-full-scan.html, zap-full-scan.xml', allowEmptyArchive: true
            }
        }
    }

    post {
        always {
            echo 'Pipeline completed.'
        }
    }
}
