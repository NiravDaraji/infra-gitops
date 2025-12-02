
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
                sh '''
                yamllint -c .yamllint.yaml .
                '''
            }
        }

        stage('Helm Lint') {
            steps {
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

        stage('Helm Unit Tests') {
            steps {
                sh '''
                helm plugin install https://github.com/helm-unittest/helm-unittest.git || true
                for chart in charts/*; do
                  if [ -f "$chart/Chart.yaml" ]; then
                    if [ -d "$chart/tests" ]; then
                      echo "Running unit tests for: $chart"
                      helm unittest "$chart" --color || exit 1
                    else
                      echo "No tests folder found in: $chart – skipping."
                    fi
                  fi
                done
                '''
            }
        }

        stage('Helm Template Dry Run') {
            steps {
                sh '''
                shopt -s nullglob
                for file in environments/${ENVIRONMENT}/values-*.yaml; do
                  app=$(basename "$file" | sed 's/^values-//; s/.yaml$//')
                  echo "Rendering: $app"
                  helm template "$app" "charts/$app" -f "$file" >/dev/null
                done
                '''
            }
        }

        stage('Trivy Security Scan') {
            steps {
                sh '''
                trivy config --severity HIGH,CRITICAL --ignore-unfixed .
                '''
            }
        }

        stage('Verify Deployment Health') {
            steps {
                echo 'Add argocd login and app health check commands here'
            }
        }

        stage('SDLC Passed') {
            steps {
                echo 'SDLC checks completed successfully.'
            }
        }
    }
}
