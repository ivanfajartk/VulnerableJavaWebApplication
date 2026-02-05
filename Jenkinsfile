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
                    reuseNode true
                }
            }
            steps {
                retry(2) {
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
                    args '-u root --entrypoint= -v ${WORKSPACE}:/zap/wrk/:rw'
                }
            }
            steps {
                retry(2) {
                    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                        // Perbaikan: Gunakan http jika aplikasi tidak menggunakan SSL
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
            // Memindahkan blok node ke dalam script agar tidak error sintaks
            node('built-in') {
                script {
                    echo 'Importing results to DefectDojo...'
                    
                    // Trufflehog Import
                    sh 'curl -X POST http://localhost:8081/api/v2/import-scan/ -H "Authorization: Token 76c969c472ea330260dd0920eed9fb29068bf044" -F "scan_type=Trufflehog Scan" -F "file=@trufflehogscan.json" -F "engagement=2"'
                    
                    // SpotBugs Import - Perbaikan Path: File ada di folder target/
                    sh 'curl -X POST http://localhost:8081/api/v2/import-scan/ -H "Authorization: Token 76c969c472ea330260dd0920eed9fb29068bf044" -F "scan_type=SpotBugs Scan" -F "file=@target/spotbugsXml.xml" -F "engagement=2"'
                    
                    // ZAP Import
                    sh 'curl -X POST http://localhost:8081/api/v2/import-scan/ -H "Authorization: Token 76c969c472ea330260dd0920eed9fb29068bf044" -F "scan_type=ZAP Scan" -F "file=@zap-full-scan.xml" -F "engagement=2"'
                }
            }
            echo 'Pipeline completed.'
        }
        cleanup {
            // Menjalankan cleanup di agent yang memiliki docker
            node('built-in') {
                sh 'docker image prune -f || true'
            }
        }
    }
}
