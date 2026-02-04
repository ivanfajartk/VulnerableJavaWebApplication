pipeline {
    agent none

    stages {
        stage('Maven Compile and SAST Spotbugs') {
            agent { label 'maven' }
            steps {
                sh 'mvn clean compile spotbugs:spotbugs -Dspotbugs.xmlOutput=true -Dspotbugs.htmlOutput=true'
                archiveArtifacts artifacts: 'target/spotbugsXml.xml, target/spotbugs.html', allowEmptyArchive: true
            }
        }

        stage('Secret Scanning') {
            agent {
                docker {
                    image 'trufflesecurity/trufflehog:latest'
                    args '--entrypoint='
                    // Menambahkan retry jika download image gagal
                    reuseNode true
                }
            }
            steps {
                retry(2) { // Mencoba ulang jika terjadi error network/download
                    script {
                        def status = sh(script: 'trufflehog filesystem . --no-update --json > trufflehogscan.json', returnStatus: true)
                        if (status != 0) echo "WARNING: Secrets found!"
                    }
                }
                archiveArtifacts artifacts: 'trufflehogscan.json', allowEmptyArchive: true
            }
        }

        stage('Build & Run App') {
            agent { label 'built-in' }
            steps {
                sh 'docker build -t vulnerable-java-application:0.1 .'
                sh 'docker rm --force vulnerable-java-application || true'
                sh 'docker run --name vulnerable-java-application -p 9000:9000 -d vulnerable-java-application:0.1'
            }
        }
        
        stage('DAST') {
            agent {
                docker {
                    image 'ghcr.io/zaproxy/zaproxy:stable'
                    // Gunakan WORKSPACE agar mounting konsisten
                    args '-u root --entrypoint= -v ${WORKSPACE}:/zap/wrk/:rw'
                }
            }
            steps {
                retry(2) {
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                        // Gunakan http (bukan https) jika aplikasi tidak pakai SSL
                        // Gunakan 172.17.0.1 (IP default Docker Bridge) jika IP container berubah-ubah
                        sh 'zap-full-scan.py -t http://172.18.0.3:9000 -r zap-full-scan.html -x zap-full-scan.xml'
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'zap-full-scan.html, zap-full-scan.xml', allowEmptyArchive: true
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline completed.'
        }
        cleanup {
            // Membersihkan image yang tidak terpakai agar disk tidak penuh
            sh 'docker image prune -f || true'
        }
    }
}
