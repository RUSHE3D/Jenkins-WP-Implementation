pipeline {
    agent {
        node {
            label 'php8'
        }
    }
    environment {
        ARM_CLIENT_ID = "XXXX"
        ARM_CLIENT_SECRET = "XXXX"
        ARM_SUBSCRIPTION_ID = "XXXX"
        ARM_TENANT_ID = "XXXX"

        AKS_CLUSTER = "XXXX-XXXX"
        AKS_RESOURCE_GROUP = "XXXX-Group"

        GIT_REPO_URL = "git@github.com:XXXX/XXXX.git"
        GIT_BRANCH = "main"
    }
    stages {  
        stage('Approval') {
            steps {
                script {
                    input message: '¿Quieres ejecutar?', submitter: 'admin'
                }
            }
        }
        stage('Checking tools') {
            steps {
                script {
                    def azResult = sh(script: 'command -v az', returnStatus: true)
                    def kubectlResult = sh(script: 'command -v kubectl', returnStatus: true)
                    def kubeloginResult = sh(script: 'command -v kubelogin', returnStatus: true)
                    if (azResult != 0) {
                        echo "Instalando AZ CLI..."
                        sh 'curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash'
                    }
                    if (kubectlResult != 0) {
                        echo "Instalando kubectl..."
                        sh 'sudo az aks install-cli'
                    }
                    if (kubeloginResult != 0) {
                        echo "kubelogin no está instalado. Instalando kubelogin..."
                        sh 'curl -sSL https://github.com/Azure/kubelogin/releases/download/v0.0.5/kubelogin-linux-amd64.zip -o kubelogin.zip'
                        sh 'unzip kubelogin.zip -d kubelogin'
                        sh 'sudo mv kubelogin/bin/linux_amd64/kubelogin /usr/local/bin/'
                    }
                    echo "AZ CLI, KUBECTL y KUBELOGIN están correctamente instalados."
                }
            }
        }
        stage('Assign Role to Service Principal') {
            when {
                expression {
                    input message: '¿Quieres asignar permisos al servicio principal?', ok: 'Asignar', submitter: 'admin'
                }
            }
            steps {
                script {
                    echo "Asignando rol 'Azure Kubernetes Service Cluster User' al servicio principal..."
                    sh '''
                        az role assignment create --assignee $ARM_CLIENT_ID --role "ROL_A_ASIGNAR" --scope /subscriptions/$ARM_SUBSCRIPTION_ID/resourceGroups/$AKS_RESOURCE_GROUP
                    '''
                    echo "Verificando asignación de rol..."
                    sh '''
                        az role assignment list --assignee $ARM_CLIENT_ID --scope /subscriptions/$ARM_SUBSCRIPTION_ID/resourceGroups/$AKS_RESOURCE_GROUP
                    '''
                }
            }
        }
        stage('Login and Config') {
            steps {
                script {
                    echo "Iniciando sesión en Azure..."
                    sh 'az login --service-principal -u $ARM_CLIENT_ID -p $ARM_CLIENT_SECRET --tenant $ARM_TENANT_ID'
                    echo "Seleccionando la suscripción..."
                    sh 'az account set --subscription $ARM_SUBSCRIPTION_ID'
                    echo "Obteniendo credenciales de AKS..."
                    sh 'az aks get-credentials --resource-group $AKS_RESOURCE_GROUP --name $AKS_CLUSTER --overwrite-existing'
                    echo "Convirtiendo kubeconfig para Azure CLI..."
                    sh 'kubelogin convert-kubeconfig -l azurecli'
                    sh 'kubectl config use-context $AKS_CLUSTER'
                }
            }
        }
        stage('Clone Repository') {
            steps {
                script {
                    echo "Clonando el repositorio..."
                    sshagent(['github-ssh-key']) {
                        sh 'rm -rf repo && git clone -b $GIT_BRANCH $GIT_REPO_URL repo'
                    }
                }
            }
        }
        stage('Apply Kubernetes Manifests') {
            steps {
                script {
                    echo "Aplicando manifiestos de Kubernetes..."
                    sh 'kubectl apply -f repo/'
                }
            }
        }
    }
}
