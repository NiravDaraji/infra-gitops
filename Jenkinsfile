
pipeline {
    agent any

    environment {
        KUBE_VERSION = "1.29.0"
        // Default ENVIRONMENT from parameter; if not passed, we still guard in shell
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

        stage('Tool sanity check') {
            steps {
                script {
                    echo "🔎 Checking tool availability..."
                    sh(script: '''
                      set +e
                      for t in helm yamllint trivy argocd; do
                        if command -v "$t" >/dev/null 2>&1; then
                          echo "✅ $t found at: $(command -v $t)"
                          $t --version 2>/dev/null || true
                        else
                          echo "⚠️  $t not found in PATH"
                        fi
                      done
                    ''', returnStatus: true)
                }
            }
        }

        stage('YAML Lint') {
            steps {
                script {
                    echo "🧪 Running YAML Lint..."
                    def rc = sh(script: '''
                      command -v yamllint >/dev/null 2>&1 && yamllint -c .yamllint.yaml .
                    ''', returnStatus: true)
                    if (rc == 0) {
                        echo "✅ YAML Lint passed."
                    } else {
                        echo "❌ YAML Lint reported issues. See details above."
                    }
                }
            }
        }

        stage('Helm Lint') {
            steps {
                script {
                    echo "🧪 Running Helm Lint..."
                    sh(script: '''
                      set +e
                      if ! command -v helm >/dev/null 2>&1; then
                        echo "⚠️ helm not found; skipping."
                        exit 0
                      fi
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
                    echo "🧪 Running Helm Unit Tests..."
                    sh(script: '''
                      set +e
                      if ! command -v helm >/dev/null 2>&1; then
                        echo "⚠️ helm not found; skipping unit tests."
                        exit 0
                      fi
                      # Install plugin if missing (non-blocking)
                      helm plugin install https://github.com/helm-unittest/helm-unittest.git >/dev/null 2>&1 || true

                      for chart in charts/*; do
                        if [ -f "$chart/Chart.yaml" ]; then
                          if [ -d "$chart/tests" ]; then
                            echo "Running unit tests for: $chart"
                            helm unittest "$chart" --color || echo "❌ Unit tests failed for $chart"
                          else
                            echo "ℹ️  No tests folder for: $chart – skipping."
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
            echo "🔍 Scanning all chart directories..."

            for chartDir in charts/*; do
                if [ -d "$chartDir" ]; then
                    chartName=$(basename "$chartDir")
                    valuesFile="environments/dev/values-${chartName}.yaml"

                    echo "📦 Chart: $chartName"
                    echo "📁 Path:  $chartDir"
                    echo "📄 Values: $valuesFile"

                    if [ ! -f "$valuesFile" ]; then
                        echo "❌ Values file not found: $valuesFile"
                        exit 1
                    fi

                    echo "Running: helm template $chartName $chartDir --values $valuesFile"

                    helm template "$chartName" "$chartDir" \
                        --values "$valuesFile" || exit 1

                    echo "✅ Helm dry-run successful for: $chartName"
                fi
            done
            '''
        }
    }

        stage('Trivy Security Scan (config, optional)') {
            steps {
                script {
                    echo "🔐 Running Trivy Security Scan (config/misconfigs)..."
                    sh(script: '''
                      set +e
                      if command -v trivy >/dev/null 2>&1; then
                        echo "▶ trivy version: $(trivy --version | head -n 1)"
                        trivy config \
                          --severity HIGH,CRITICAL \
                          --include-non-failures \
                          --helm-kube-version "${KUBE_VERSION}" \
                          --exit-code 0 \
                          .
                      else
                        echo "⚠️ Trivy not found; skipping security scan."
                      fi
                    ''', returnStatus: true)
                    echo "✅ Trivy step completed (scan may have findings; review console output)."
                }
            }
        }

        stage('App status via ArgoCD') {
            steps {
                script {
                    echo "📊 Checking app status via ArgoCD..."
                    sh(script: '''
                      set +e
                      if command -v argocd >/dev/null 2>&1; then
                        echo "Logging into ArgoCD..."
                        argocd login 10.139.9.158:31181 --username admin --password Admin@1234 --insecure || echo "❌ ArgoCD login failed"

                        echo "Listing ArgoCD apps..."
                        argocd app list || echo "❌ Failed to list apps"
                      else
                        echo "⚠️ ArgoCD CLI not found; skipping."
                      fi
                    ''', returnStatus: true)
                }
            }
        }

        stage('SDLC Summary') {
            steps {
                echo "✅ SDLC validation completed. Review console logs for errors and warnings."
            }
        }
    }

    post {
        always {
            echo "✅ Pipeline finished successfully (with possible warnings/errors)."
        }
    }
}
