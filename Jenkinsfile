pipeline {
  agent any

  stages {
   








   stage('Build Artifact') {
            steps {
              sh "mvn clean package -DskipTests=true"

              archive 'target/*.jar' //so that 
            }
        }  




 stage('Mutation Tests - PIT') {
  	steps {
    	sh "mvn org.pitest:pitest-maven:mutationCoverage"
  	}
    	post {
     	always {
       	pitmutation mutationStatsFile: '**/target/pit-reports/**/mutations.xml'
     	}
   	}
	}





			stage('Vulnerability Scan - Docker Trivy') {
			steps {
				withCredentials([string(credentialsId: 'trivy_sabrine', variable: 'TOKEN')]) {
					catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
						sh "sed -i 's#token_github#${TOKEN}#g' trivy-image-scan.sh" 	 
						sh "sudo bash trivy-image-scan.sh"

					}
			}
			}
			}





		stage('SONAR SCAN  ') {
					steps {
			
			                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {

								withSonarQubeEnv('SonarQube') {
									sh "mvn sonar:sonar \
  -Dsonar.projectKey=sabrine \
  -Dsonar.projectName=sabrine \
  -Dsonar.host.url=http://mytpm.eastus.cloudapp.azure.com:9999"

								}

				
							}


				}

		}





 stage('Docker Build and Push') {
  	steps {
    	withCredentials([string(credentialsId: 'dockerhub', variable: 'DOCKER_HUB_PASSWORD')]) {
      	sh 'sudo docker login -u sabrine24 -p $DOCKER_HUB_PASSWORD'
      	sh 'printenv'
      	sh 'sudo docker build -t sabrine24/devops-app:""$GIT_COMMIT"" .'
      	sh 'sudo docker push sabrine24/devops-app:""$GIT_COMMIT""'
    	}

  	}
	}



 stage('Deployment Kubernetes ') {
  	steps {
    	withKubeConfig([credentialsId: 'kubeconfig']) {
           	sh "sed -i 's#replace#sabrine24/devops-app:${GIT_COMMIT}#g' k8s_deployment_service.yaml"
           	sh "kubectl apply -f k8s_deployment_service.yaml"
         	}
  	}

	}



 stage('Vulnerability Scan - Docker') {
   steps {
    	catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
			 sh "mvn dependency-check:check"
    	}
   	 }
   	 post {
  	always {
   			 dependencyCheckPublisher pattern: 'target/dependency-check-report.xml'
   			 }
   	 }
 }


 stage('Vulnerability Scan - Kubernetes') {
   	steps {
     	parallel(
       	"OPA Scan": {
         	sh 'sudo docker run --rm -v $(pwd):/project openpolicyagent/conftest test --policy opa-k8s-security.rego k8s_deployment_service.yaml'
       	},
       	"Kubesec Scan": {
         	sh "sudo bash kubesec-scan.sh"
       	},
       	"Trivy Scan": {
         	sh "sudo bash trivy-k8s-scan.sh"
       	}
     	)
   	}
 	}


 stage('OWASP ZAP - DAST') {
   	steps {
     	withKubeConfig([credentialsId: 'kubeconfig']) {
       	sh 'sudo bash zap.sh'
     	}
   	}
 	}



stage('Integration Tests - DEV') {
           steps {
             script {
               try {
                 withKubeConfig([credentialsId: 'kubeconfig']) {
                   sh "bash integration-test.sh"
                 }
               } catch (e) {
                 withKubeConfig([credentialsId: 'kubeconfig']) {
                   sh "kubectl -n default rollout undo deploy ${deploymentName}"
                 }
                 throw e
               }
             }
           }
         }












  
	}
}