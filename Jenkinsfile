pipeline {
	agent none
	environment {
		DOCKERHUB_CREDENTIALS = credentials('DockerLogin')
	}
	stages {
		stage('Secret Scanning Using Trufflehog'){
			agent {
				docker {
					image 'trufflesecurity/trufflehog:latest'
					args '--entrypoint='
				}
			}

			steps {
				catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
					sh'trufflehog filesystem . --exclude-paths trufflehog-excluded-paths.txt --fail --json --no-update > trufflehog-scan-result.json'
				}
				sh 'cat trufflehog-scan-result.json'
				archiveArtifacts artifacts: 'trufflehog-scan-result.json'
			}
		}
		stage('Build') {
			agent {
				docker {
				image 'node:lts-buster-slim'
				}
			}
			steps {
				sh 'npm install'
			}
		}
		stage('Build Docker Image and Push to Docker Registry') {
			agent {
				docker {
					image 'docker:dind'
					args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
				}
			}
			steps {
				sh 'docker build -t sanzudock/nodejsgoof:0.1 .'
				sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
				sh 'docker push sanzudock/nodejsgoof:0.1'
				}
			}
		stage('Deploy Docker Image') {
			agent {
				docker {
					image 'kroniak/ssh-client'
					args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
				}
			}
			steps {
				withCredentials([sshUserPrivateKey(credentialsId: "DeploymentSSHkey", keyFileVariable:'keyfile')]) {
					sh 'ssh -i ${keyfile} -o StrictHostKeyChecking=no deployment@192.168.1.13 "echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin"'
					sh 'ssh -i ${keyfile} -o StrictHostKeyChecking=no deployment@192.168.1.13 docker pull sanzudock/nodejsgoof:0.1'
					sh 'ssh -i ${keyfile} -o StrictHostKeyChecking=no deployment@192.168.1.13 docker rm --force mongodb'
					sh 'ssh -i ${keyfile} -o StrictHostKeyChecking=no deployment@192.168.1.13 docker run --detach --name mongodb -p 27017:27017 mongo:3'
					sh 'ssh -i ${keyfile} -o StrictHostKeyChecking=no deployment@192.168.1.13 docker rm --force nodejsgoof'
					sh 'ssh -i ${keyfile} -o StrictHostKeyChecking=no deployment@192.168.1.13 docker run -it --detach --name nodejsgoof --network host sanzudock/nodejsgoof:0.1'
				}
			
			}
		}
	}
}
