pipeline {
	agent none
	stages {
		stage('Maven Compile') {
			agent {
				label 'maven'
			}
			steps {
				sh 'mvn compile'
			}	
		}
		stage ('Build Docker Image') {
		    agent {
		        docker {
		            image 'docker:dind'
		            args '-u root -v /var/run/docker.sock:/var/run/docker.sock'
		        }
			}
			steps {
				sh 'docker build -t vulnerable-java-application:0.1 .'
			}	
		}
		stage ('Run Docker Image') {
			agent {
				label 'built-in'
			}
			steps {
				sh 'docker rm --force vulnerable-java-application'
				sh 'docker run --name vulnerable-java-application -p 9000:9000 -d vulnerable-java-application:0.1'
			}
		}
	}
}	
