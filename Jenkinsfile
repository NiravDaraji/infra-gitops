
pipeline {
    agent any

    environment {
        ENVIRONMENT  = "${params.environment ?: 'dev'}"
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
                    def rc = sh(script: '''
                      yamllint -c .yamllint.yaml environments/dev/
                    ''', returnStatus: true)
                    if (rc == 0) {
                        echo "YAML Lint passed."
                    } else {
                        echo "YAML Lint reported issues. See details above."
                    }
                }
            }
        }

        stage('Helm Lint') {
            steps {
                script {
                    echo "Running Helm Lint..."
                    sh(script: '''
                      set +e
                      for chart in charts/*; do
                        if [ -f "$chart/Chart.yaml" ]; then
                          echo "Linting $chart..."
                          helm lint "$chart" || echo "Helm lint failed for $chart"
                        fi
                      done
                    ''', returnStatus: true)
                }
            }
        }

        stage('Helm Unit Tests') {
            steps {
                script {
                    echo "Running Helm Unit Tests..."
                    sh(script: '''
                      set +e
                      helm plugin install https://github.com/helm-unittest/helm-unittest.git >/dev/null 2>&1 || true
                      for chart in charts/*; do
                        if [ -f "$chart/Chart.yaml" ]; then
                          if [ -d "$chart/tests" ]; then
                            echo "Running unit tests for: $chart"
                            helm unittest "$chart" || echo "Unit tests failed for $chart"
                          else
                            echo "No tests folder for: $chart – skipping."
                          fi
                        fi
                      done
                    ''', returnStatus: true)
                }
            }
        }

         stage('Helm Template Dry Run For All Charts') {
        steps {
            sh '''
            echo "Scanning all chart directories..."

            for chartDir in charts/*; do
                if [ -d "$chartDir" ]; then
                    chartName=$(basename "$chartDir")
                    valuesFile="environments/dev/values-${chartName}.yaml"

                    echo "Chart: $chartName"
                    echo "Path:  $chartDir"
                    echo "Values: $valuesFile"

                    if [ ! -f "$valuesFile" ]; then
                        echo "Values file not found: $valuesFile"
                        exit 1
                    fi

                    echo "Running: helm template $chartName $chartDir --values $valuesFile"

                    helm template "$chartName" "$chartDir" \
                        --values "$valuesFile" || exit 1

                    echo "Helm dry-run successful for: $chartName"
                fi
            done
            '''
        }
    }

        stage('Trivy Security Scan (config, optional)') {
            steps {
                script {
                    echo "Running Trivy Security Scan..."
                    sh(script: '''
                      set +e
                      trivy config \
                        --severity HIGH,CRITICAL \
                        --include-non-failures \
                        --exit-code 0 \
                        .
                    ''', returnStatus: true)
                    echo "Trivy step completed."
                }
            }
        }

        stage('App status via ArgoCD') {
            steps {
                script {
                    echo "Checking app status via ArgoCD..."
                    sh(script: '''
                      set +e
                      argocd login 10.139.9.158:31181 --username admin --password Admin@1234 --insecure || echo "❌ ArgoCD login failed"
                      argocd app list || echo "Failed to list apps"
                    ''', returnStatus: true)
                }
            }
        }

        stage('SDLC Summary') {
            steps {
                echo "SDLC validation completed. Review console logs for errors and warnings."
            }
        }
    }

    post {
        always {
            echo "Pipeline finished successfully (with possible warnings/errors)."
        }
    }
}
