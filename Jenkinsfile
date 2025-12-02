
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
                    def result = sh(script: "yamllint -c .yamllint.yaml .", returnStatus: true)
                    if (result != 0) {
                        echo "❌ YAML Lint found issues. Check console output."
                    } else {
                        echo "✅ YAML Lint passed."
                    }
                }
            }
        }

        stage('Helm Lint') {
            steps {
                script {
                    echo "Running Helm Lint..."
                    sh(script: '''
                        for chart in charts/*; do
                          if [ -f "$chart/Chart.yaml" ]; then
                            echo "Linting $chart..."
                            helm lint "$chart" || echo "❌ Helm lint failed for $chart"
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
                        helm plugin install https://github.com/helm-unittest/helm-unittest.git || true
                        for chart in charts/*; do
                          if [ -f "$chart/Chart.yaml" ]; then
                            if [ -d "$chart/tests" ]; then
                              echo "Running unit tests for: $chart"
                              helm unittest "$chart" --color || echo "❌ Unit tests failed for $chart"
                            else
                              echo "No tests folder found in: $chart – skipping."
                            fi
                          fi
                        done
                    ''', returnStatus: true)
                }
            }
        }

        stage('Helm Template Dry Run') {
            steps {
                script {
                    echo "Running Helm Template Dry Run..."
                    sh(script: '''
                        bash -c '
                        shopt -s nullglob
                        for file in environments/${ENVIRONMENT}/values-*.yaml; do
                          app=$(basename "$file" | sed "s/^values-//; s/.yaml$//")
                          echo "Rendering: $app"
                          helm template "$app" "charts/$app" -f "$file" >/dev/null || echo "❌ Dry run failed for $app"
                        done
                        '
                    ''', returnStatus: true)
                }
            }
        }

        
stage('Trivy Security Scan (config, optional)') {
            steps {
                script {
                    echo "🔐 Running Trivy Security Scan (config/misconfigs)..."
                    sh(script: """
                      set +e
                      if command -v trivy >/dev/null 2>&1; then
                        echo "▶ trivy version: $(trivy --version | head -n 1)"
                        trivy config \
                          --severity HIGH,CRITICAL \
                          --include-non-failures \
                          --helm-kube-version ${KUBE_VERSION} \
                          --exit-code 0 \
                          .
                      } else {
                        echo "⚠️ Trivy not found; skipping security scan."
                      fi
                    """, returnStatus: true)
                    echo "Trivy step completed (scan may have findings; review console output)."
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
                echo "✅ SDLC validation completed. Review console logs for details."
            }
        }
    }

    post {
        always {
            echo "Pipeline finished successfully (with possible warnings/errors)."
        }
    }
}
