pipeline {
	agent any
	tools {
		nodejs 'NodeJS'
	}
	environment {
		DOCKER_HUB_REPO = 'asifniloy45/gitops-app'
		DOCKER_HUB_CREDENTIALS_ID = 'gitops-dockerhub'
	}
	stages {
		stage('Checkout Github'){
			steps {
			git branch: 'main', credentialsId: 'Jenkins-GitOps-token-Github', url: 'https://github.com/asifichy/Jenkins-ArgoCD-GitOps.git'
			}
		}		
		stage('Install node dependencies'){
			steps {
				sh 'npm install'
			}
		}
		stage('Build Docker Image'){
			steps {
				script {
					echo 'building docker image...'
					dockerImage = docker.build("${DOCKER_HUB_REPO}:latest")
				}
			}
		}
		stage('Trivy Scan'){
			steps {
				sh 'trivy --severity HIGH,CRITICAL --no-progress image --format table -o trivy-scan-report.txt ${DOCKER_HUB_REPO}:latest'
				sh 'trivy --severity HIGH,CRITICAL --skip-update --no-progress image --format table -o trivy-scan-report.txt ${DOCKER_HUB_REPO}:latest'
			}
		}
		stage('Push Image to DockerHub'){
			steps {
				script {
					echo 'pushing docker image to DockerHub...'
					docker.withRegistry('https://registry.hub.docker.com', "${DOCKER_HUB_CREDENTIALS_ID}"){
						dockerImage.push('latest')
						}
					}
				}
			}
		stage('Install Kubectl & ArgoCD CLI'){
			steps {
				sh '''
				echo 'installing Kubectl & ArgoCD cli...'
				curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
				chmod +x kubectl
				mv kubectl /usr/local/bin/kubectl
				curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
				chmod +x /usr/local/bin/argocd
				'''
			}
		}
		stage('Configure ArgoCD RBAC') {
			steps {
				sh '''
				# Create ArgoCD role binding for Jenkins
				kubectl create rolebinding jenkins-argocd \
				--clusterrole=admin \
				--serviceaccount=default:default \
				--namespace=argocd || true
				'''
			}
		}
		stage('Apply Kubernetes Manifests & Sync App with ArgoCD'){
			steps {
				script {
					kubeconfig(credentialsId: 'kubeconfig', serverUrl: 'https://192.168.49.2:8443') {
    						sh '''
							ARGOCD_PASSWORD=$(kubectl get secret -n argocd argocd-initial-admin-secret \
							-o jsonpath="{.data.password}" | base64 -d)
							
							argocd login 54.179.152.192:31354 \
							--username admin \
							--password $ARGOCD_PASSWORD \
							--grpc-web \
							--insecure
							argocd app sync jenkins-argocd-gitops --retry-limit 5 --retry-backoff-duration 30s
						'''
					}	
				}
			}
		}
		stage('Verify ArgoCD Connectivity') {
			steps {
				sh '''
				# Check if ArgoCD server is reachable
				timeout 30 bash -c 'until nc -zv 54.179.152.192 31354; do sleep 2; done'
				
				# Verify API access
				argocd version --client
				'''
			}
		}
	}

	post {
		success {
			echo 'Build & Deploy completed succesfully!'
		}
		failure {
			echo 'Build & Deploy failed. Check logs.'
		}
	}
}
