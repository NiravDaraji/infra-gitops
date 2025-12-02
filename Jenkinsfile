
pipeline {
    agent any

    environment {
        KUBE_VERSION = "1.29.0"
        ENVIRONMENT = "${params.environment ?: 'dev'}"
    }

    parameters {
        string(name: 'environment', defaultValue: 'dev', description: 'Environment to validate')
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'dev', url: 'https://github.com/NiravDaraji/infra-gitops.git'
            }
        }

        stage('YAML Lint') {
            steps {
                script {
                    echo "Running YAML Lint..."
                    def result = sh(script: "yamllint -c .yamllint.yaml . || true", returnStatus: true)
                    if (result != 0) {
                        echo "YAML Lint completed with errors. Check console output."
                    }
                }
            }
        }

        stage('Helm Lint') {
            steps {
                script {
                    echo "Running Helm Lint..."
                    sh '''
                    for chart in charts/*; do
                      if [ -f "$chart/Chart.yaml" ]; then
                        echo "Linting $chart..."
                        helm lint "$chart" || echo "Helm lint failed for $chart"
                      fi
                    done
                    '''
                }
            }
        }

        stage('Helm Unit Tests') {
            steps {
                script {
                    echo "Running Helm Unit Tests..."
                    sh '''
                    helm plugin install https://github.com/helm-unittest/helm-unittest.git || true
                    for chart in charts/*; do
                      if [ -f "$chart/Chart.yaml" ]; then
                        if [ -d "$chart/tests" ]; then
                          echo "Running unit tests for: $chart"
                          helm unittest "$chart" --color || echo "Unit tests failed for $chart"
                        else
                          echo "No tests folder found in: $chart – skipping."
                        fi
                      fi
                    done
                    '''
                }
            }
        }

        stage('Helm Template Dry Run') {
            steps {
                script {
                    echo "Running Helm Template Dry Run..."
                    sh '''
                    shopt -s nullglob
                    for file in environments/${ENVIRONMENT}/values-*.yaml; do
                      app=$(basename "$file" | sed 's/^values-//; s/.yaml$//')
                      echo "Rendering: $app"
                      helm template "$app" "charts/$app" -f "$file" >/dev/null || echo "Dry run failed for $app"
                    done
                    '''
                }
            }
        }

        stage('Trivy Security Scan') {
            steps {
                script {
                    echo "Running Trivy Security Scan..."
                    sh '''
                    trivy config --severity HIGH,CRITICAL --ignore-unfixed . || echo "Trivy scan found issues"
                    '''
                }
            }
        }

        stage('Verify Deployment Health') {
            steps {
                echo "Add ArgoCD health check commands here (non-blocking)"
            }
        }

        stage('SDLC Summary') {
            steps {
                echo "SDLC validation completed. Review console logs for details."
            }
        }
    }

    post {
        always {
            echo "Pipeline finished. Some checks may have warnings or errors."
        }
    }
}
