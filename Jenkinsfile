pipeline {
    agent {
        kubernetes {
            yaml '''
                apiVersion: v1
                kind: Pod
                metadata:
                  labels:
                    jenkins: agent
                spec:
                  serviceAccountName: jenkins-sa
                  containers:
                  - name: helm
                    image: alpine/helm:3.12.0
                    command:
                    - cat
                    tty: true
                  - name: vault
                    image: hashicorp/vault:1.13.0
                    command:
                    - cat
                    tty: true
                    env:
                    - name: VAULT_ADDR
                      value: http://vault:8200
            '''
        }
    }

    environment {
        // Bitbucket credentials
        BITBUCKET_CREDS = credentials('bitbucket-credentials')
        
        // Vault configuration
        VAULT_ADDR = "http://vault:8200"
        VAULT_SECRET_PATH = "secret/data/superset"
        VAULT_ROLE = "superset"
        VAULT_AUTH_PATH = "auth/kubernetes"
        K8S_NAMESPACE = "superset"
        SERVICE_ACCOUNT = "superset"
        
        // Kubernetes configuration
        KUBECONFIG = credentials('kubeconfig')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Configure Vault') {
            steps {
                container('vault') {
                    script {
                        // Получаем токен для доступа к Vault через Kubernetes auth
                        def vaultToken = sh(
                            script: '''
                            JWT=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
                            curl --request POST \
                                --data "{\"jwt\": \"$JWT\", \"role\": \"${VAULT_ROLE}\"}" \
                                ${VAULT_ADDR}/v1/auth/${VAULT_AUTH_PATH}/login | jq -r '.auth.client_token'
                            ''',
                            returnStdout: true
                        ).trim()
                        
                        // Сохраняем токен для использования в следующих шагах
                        env.VAULT_TOKEN = vaultToken
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                container('helm') {
                    script {
                        // Добавляем необходимые репозитории
                        sh "helm repo add bitnami https://charts.bitnami.com/bitnami"
                        sh "helm repo update"
                        
                        // Создаем namespace если он не существует
                        sh "kubectl create namespace ${K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -"
                        
                        // Создаем сервисный аккаунт если он не существует
                        sh """
                        kubectl create serviceaccount ${SERVICE_ACCOUNT} \
                            --namespace ${K8S_NAMESPACE} \
                            --dry-run=client -o yaml | kubectl apply -f -
                        """
                        
                        // Устанавливаем/обновляем Helm чарт
                        sh """
                        helm upgrade --install my-superset ./superset \
                            --namespace ${K8S_NAMESPACE} \
                            --set vault.addr=${VAULT_ADDR} \
                            --set vault.role=${VAULT_ROLE} \
                            --set vault.secretPath=${VAULT_SECRET_PATH} \
                            --set vault.authPath=${VAULT_AUTH_PATH} \
                            --wait --timeout 10m
                        """
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                container('helm') {
                    script {
                        // Проверяем статус деплоймента
                        sh """
                        kubectl rollout status deployment/my-superset \
                            --namespace ${K8S_NAMESPACE} \
                            --timeout=300s
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Deployment successful!'
        }
        failure {
            echo 'Deployment failed!'
        }
    }
} 