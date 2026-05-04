pipeline {
    agent any

    tools {
        nodejs 'nodejs'
    }

    environment {
        ACR_NAME = 'dheepuacr2026'
        ACR_LOGIN_SERVER = 'dheepuacr2026.azurecr.io'
        IMAGE_NAME = 'nodejs-shopping-cart'
        RESOURCE_GROUP = 'aks-rg'
        AKS_CLUSTER = 'dheepu-aks-cluster2026'
        HELM_RELEASE = 'nodejs-shopping-cart'
        HELM_CHART_PATH = 'helm/nodejs-shopping-cart'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'master', url: 'https://github.com/ravindrandheepika/nodejs-shopping-cart'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                node -v
                npm -v
                npm install
                '''
            }
        }

        stage('SonarQube SAST Scan') {
            steps {
                script {
                    def scannerHome = tool 'sonar-scanner'

                    withSonarQubeEnv('SonarQube') {
                        withCredentials([string(credentialsId: 'SONAR_AUTH_TOKEN', variable: 'SONAR_AUTH_TOKEN')]) {
                            sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=nodejs-shopping-cart-dheepu \
                            -Dsonar.projectName=nodejs-shopping-cart \
                            -Dsonar.sources=. \
                            -Dsonar.exclusions=node_modules/**,helm/**,data/** \
                            -Dsonar.token=$SONAR_AUTH_TOKEN
                            """
                        }
                    }
                }
            }
        }

        stage('Snyk SCA Scan') {
            steps {
                withCredentials([string(credentialsId: 'SNYK_TOKEN', variable: 'SNYK_TOKEN')]) {
                    sh '''
                    npm install -g snyk
                    snyk auth $SNYK_TOKEN
                    snyk test --severity-threshold=high || true
                    snyk monitor || true
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                docker build -t $IMAGE_NAME:$IMAGE_TAG .
                docker tag $IMAGE_NAME:$IMAGE_TAG $ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG
                docker tag $IMAGE_NAME:$IMAGE_TAG $ACR_LOGIN_SERVER/$IMAGE_NAME:latest
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                trivy image --severity HIGH,CRITICAL --exit-code 0 $ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG
                '''
            }
        }

        stage('Login to Azure & ACR') {
            steps {
                withCredentials([
                    usernamePassword(credentialsId: 'azure-sp', usernameVariable: 'AZURE_APP_ID', passwordVariable: 'AZURE_PASSWORD'),
                    string(credentialsId: 'azure-tenant-id', variable: 'AZURE_TENANT')
                ]) {
                    sh '''
                    az login --service-principal -u $AZURE_APP_ID -p $AZURE_PASSWORD --tenant $AZURE_TENANT
                    az acr login --name $ACR_NAME
                    '''
                }
            }
        }

        stage('Push Image to ACR') {
            steps {
                sh '''
                docker push $ACR_LOGIN_SERVER/$IMAGE_NAME:$IMAGE_TAG
                docker push $ACR_LOGIN_SERVER/$IMAGE_NAME:latest
                '''
            }
        }

        stage('Connect to AKS') {
            steps {
                sh '''
                az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER --overwrite-existing
                '''
            }
        }

        stage('Helm Deploy to AKS') {
            steps {
                sh '''
                helm upgrade --install $HELM_RELEASE $HELM_CHART_PATH \
                --set image.repository=$ACR_LOGIN_SERVER/$IMAGE_NAME \
                --set image.tag=$IMAGE_TAG
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                kubectl get pods -o wide
                kubectl get svc
                kubectl get hpa
                '''
            }
        }

        stage('Check LoadBalancer Status') {
            steps {
                sh '''
                echo "Checking LoadBalancer status..."
                kubectl get svc nodejs-shopping-cart-service -o wide
                echo "Describing service events..."
                kubectl describe svc nodejs-shopping-cart-service | grep -A5 Events || true
                '''
            }
        }

        stage('Wait for External IP') {
            steps {
                sh '''
                echo "Waiting for LoadBalancer EXTERNAL-IP..."
                for i in {1..10}; do
                  IP=$(kubectl get svc nodejs-shopping-cart-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
                  if [ -n "$IP" ]; then
                    echo "External IP assigned: $IP"
                    exit 0
                  else
                    echo "Still pending... retrying in 30 seconds"
                    sleep 30
                  fi
                done

                echo "ERROR: LoadBalancer did not receive an external IP in time."
                exit 1
                '''
            }
        }
    }
}
