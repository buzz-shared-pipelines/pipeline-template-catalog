pipeline {
  agent none
  options { 
    buildDiscarder(logRotator(numToKeepStr: '5'))
    skipDefaultCheckout true
  }
  stages {
    stage('Acquiring pod') {
	 agent {
            kubernetes {
                label 'node'
                yamlFile 'pod-templates/node-pod.yaml'
                }
            }	
            stages{
                stage('Install'){
                    steps {
                        container('node') {
                        checkout scm
                        sh 'npm install'
		                stash includes: 'Dockerfile', name: 'Dockerfile'
                        script{
		                version = sh(returnStdout: true, script: """
                        node -pe "require('./package.json').version" | tr -d '[:space:]'
                        """)
 		                }
                         echo "App Version: ${version}"
                   }

              }
                    }
                
            
            stage('Test'){
                    steps {
                        container('node') {
                        sh 'CI=true npm test'
                    }
                    script{
                   commitHash = sh(returnStdout: true, script: "git rev-parse HEAD | cut -c1-7 | tr -d '\n'")
                 }
            }
            post {
                always {
                   echo "done"     
                }
                }
            }

	    }
        }
        stage('Build Docker Image') {
         agent {
            kubernetes {
                label 'kaniko'
                yamlFile 'pod-templates/kaniko-pod.yaml'
                }
            }
            steps {
              container(name: 'kaniko', shell: '/busybox/sh') {
                unstash 'Dockerfile'
                withEnv(['PATH+EXTRA=/busybox:/kaniko']) {
            	sh """#!/busybox/sh
            	executor -f ${pwd()}/Dockerfile -c git://github.com/buzz-microservices/frontend-todo.git -d gcr.io/na-csa-msuarez/frontend-todo:${BUILD_NUMBER} -d gcr.io/na-csa-msuarez/frontend-todo:${version}-${commitHash} -d gcr.io/na-csa-msuarez/frontend-todo:latest
                """
                   }
              }
            }
        }
        stage('Helm install') {
         agent {
            kubernetes {
                label 'helm-kubectl'
                yamlFile 'pod-templates/helm-kubectl-pod.yaml'
                }
            }
            steps {
                checkout scm
              container('helm-kubectl') {
                sh "curl -X DELETE http://34.67.152.26:8080/api/charts/frontend-todo/0.0.1"
		        sh "helm package --version 0.0.1 --app-version ${version} frontend-todo --debug --save=false"
		        sh """curl -L --data-binary "@frontend-todo-0.0.1.tgz" http://34.67.152.26:8080/api/charts"""
		        sh "helm init --client-only"
		        sh "helm repo add chartmuseum http://34.67.152.26:8080"
                sh "helm upgrade frontend-todo chartmuseum/frontend-todo -i --namespace cje --set image.tag=${version}-${commitHash}"
                   }
            }
              }
            
        
    }
}
