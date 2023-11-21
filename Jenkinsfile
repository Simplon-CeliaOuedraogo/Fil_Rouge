pipeline {
    agent any
    
    stages {

        stage('Cloning the git') {
            steps {
                script {
                    // Define the repository URL and the target directory
                    def repoUrl = 'https://github.com/Simplon-CeliaOuedraogo/Fil_Rouge.git'
                    // Check if the current directory is a Git repository
                    if (!fileExists('.git')) {
                        // Not a Git repository, so clone the repo
                        sh "git clone ${repoUrl} ."
                    } else {
                        // Current directory is a Git repository, so pull the latest changes
                        sh('''
                            git checkout main
                            git pull
                        ''')
                    }
                }
            }
        }
        
        stage('Set up infrastructure with terraform') {
            steps {
                script {
                    withCredentials([azureServicePrincipal(credentialsId: 'ServicePrincipal')]) {
                        sh "az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET -t $AZURE_TENANT_ID"

                        // Initialize Terraform
                        dir('terraform') {
                            sh 'terraform init'

                            // Check if the AKS cluster exists in the Terraform state
                            def aksExists = sh(script: "terraform state list azurerm_kubernetes_cluster.aks", returnStatus: true) == 0

                            if (aksExists) {
                                // If AKS exists, apply changes only to the AKS cluster
                                echo "AKS cluster exists. Applying changes to AKS only."
                                sh 'terraform apply -auto-approve -target=azurerm_kubernetes_cluster.aks'
                            } else {
                                // If AKS does not exist, apply changes to the entire infrastructure
                                echo "AKS cluster does not exist. Applying changes to the entire infrastructure."
                                sh 'terraform apply -auto-approve'
                            }
                        }
                    }
                }
            }
        }

        stage('Set up Traefik ingress') {
            steps {
                script {
                    sh "az aks get-credentials -g project_celia -n cluster-project"
                    // Check if the Traefik release is already deployed
                    def isTraefikDeployed = sh(script: "helm list --namespace default -q | grep -w traefik", returnStatus: true) == 0

                    if (!isTraefikDeployed) {
                        // If the release is not deployed, install it
                        echo "Traefik is not installed. Installing Traefik."
                        sh('''
                            helm repo add traefik https://traefik.github.io/charts
                            helm repo update
                            helm install traefik traefik/traefik
                        ''')
                    } else {
                        echo "Traefik is already installed."
                    }
                }
            }
        }

        stage('Set up Wordpress with kubernetes/helm') {
            steps {
                script {
                    sh "az aks get-credentials -g project_celia -n cluster-project"
                    dir('terraform/helm') {
                        // Check if the Helm release is already deployed
                        def isDeployed = sh(script: "helm list --namespace default -q | grep -w myblog", returnStatus: true) == 0

                        if (isDeployed) {
                            // If the release is deployed, upgrade it
                            echo "Release 'myblog' exists. Upgrading chart."
                            sh 'helm upgrade myblog -f values.yaml oci://registry-1.docker.io/bitnamicharts/wordpress'
                        } else {
                            // If the release is not deployed, install it
                            echo "Release 'myblog' does not exist. Installing chart."
                            sh 'helm install myblog -f values.yaml oci://registry-1.docker.io/bitnamicharts/wordpress'
                        }

                        sh "kubectl apply -f autoscaler.yaml"
                    }
                }
            }
        }
    }
}
