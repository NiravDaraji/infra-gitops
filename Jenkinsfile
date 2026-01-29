pipeline {
    agent any

    environment {
        ENVIRONMENT         = "${params.environment ?: 'dev'}"
        PROMOTION_ELIGIBLE  = "false"
    }

    parameters {
        string(name: 'environment', defaultValue: 'dev', description: 'Environment to validate (e.g., dev)')
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'dev', url: 'https://github.com/NiravDaraji/infra-gitops.git'
            }
        }

        stage('Install Missing Tools') {
            steps {
                script {
                    echo "Checking and installing required tools..."
                    sh '''
                      set -e
                      for t in helm yamllint trivy argocd; do
                        if command -v "$t" >/dev/null 2>&1; then
                          echo "$t found at: $(command -v $t)"
                          $t --version 2>/dev/null || true
                        else
                          echo "$t not found. Installing..."
                          case "$t" in
                            helm)
                              curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
                              ;;
                            yamllint)
                              apt update && apt install -y yamllint
                              ;;
                            trivy)
                              curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh
                              ;;
                            argocd)
                              curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
                              chmod +x /usr/local/bin/argocd
                              ;;
                          esac
                          echo "$t installed successfully."
                        fi
                      done
                    '''
                }
            }
        }

        stage('YAML Lint') {
            steps {
                script {
                    echo "Running YAML Lint..."
                    sh 'yamllint -c .yamllint.yaml environments/dev/'
                }
            }
        }

        stage('Helm Lint') {
            steps {
                sh '''
                  for chart in charts/*; do
                    if [ -f "$chart/Chart.yaml" ]; then
                      echo "Linting $chart..."
                      helm lint "$chart"
                    fi
                  done
                '''
            }
        }

        stage('Helm Unit Tests') {
            steps {
                sh '''
                  helm plugin install https://github.com/helm-unittest/helm-unittest.git >/dev/null 2>&1 || true
                  for chart in charts/*; do
                    if [ -f "$chart/Chart.yaml" ] && [ -d "$chart/tests" ]; then
                      echo "Running unit tests for: $chart"
                      helm unittest "$chart"
                    fi
                  done
                '''
            }
        }

        stage('Helm Template Dry Run For All Charts') {
            steps {
                sh '''
                for chartDir in charts/*; do
                    chartName=$(basename "$chartDir")
                    valuesFile="environments/dev/values-${chartName}.yaml"

                    echo "Running helm template for $chartName"

                    if [ ! -f "$valuesFile" ]; then
                        echo "❌ Missing values file: $valuesFile"
                        exit 1
                    fi

                    helm template "$chartName" "$chartDir" \
                        --values "$valuesFile"
                done
                '''
            }
        }

        stage('Trivy Security Scan (config, optional)') {
            steps {
                sh '''
                  trivy config \
                    --severity HIGH,CRITICAL \
                    --include-non-failures \
                    --exit-code 0 \
                    .
                '''
            }
        }

        stage('App Status via ArgoCD') {
            steps {
                sh '''
                  argocd login 10.139.9.158:31181 \
                    --username admin \
                    --password Admin@1234 \
                    --insecure

                  argocd app list
                '''
            }
        }

        stage('SDLC Summary') {
            steps {
                script {
                    echo "========================================="
                    echo " SDLC WORKFLOW STATUS : PASSED"
                    echo "========================================="
                    echo "✔ YAML Validation"
                    echo "✔ Helm Lint"
                    echo "✔ Helm Unit Tests"
                    echo "✔ Helm Template Dry Run"
                    echo "✔ Trivy Security Scan (Optional)"
                    echo "✔ ArgoCD Application Status Check"
                    echo "-----------------------------------------"
                    echo "Your SDLC workflow is validated."
                    echo "You are eligible to promote this build to STAGING."
                    echo "========================================="

                    env.PROMOTION_ELIGIBLE = "true"
                }
            }
        }

        stage('Promotion Approval') {
            when {
                expression { env.PROMOTION_ELIGIBLE == "true" }
            }
            steps {
                input message: 'SDLC validated successfully. Do you want to promote this build to STAGING?',
                      ok: 'Yes, Promote'
            }
        }

        stage('Trigger Promotion Workflow') {
            when {
                expression { env.PROMOTION_ELIGIBLE == "true" }
            }
            steps {
                echo "Promotion approved."
                echo "Triggering DEV → STAGING promotion workflow..."
                echo "Here Jenkins/GitHub Action will sync tested values to staging branch."
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully."
        }
        aborted {
            echo "Pipeline aborted by user during promotion approval."
        }
        failure {
            echo "Pipeline failed. Promotion not allowed."
        }
    }
}
