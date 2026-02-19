pipeline {
    agent any

    environment {
        PROMOTION_ELIGIBLE  = "false"
    }

    parameters {
        string(
            name: 'environment',
            defaultValue: 'dev',
            description: 'Environment to validate (e.g. dev)'
        )
        choice(
            name: 'chartName',
            choices: ['openspeedtest', 'thanos', 'wordpress', 'grafana', 'speedtest', 'test_repo'],
            description: 'Select chart to validate'
        )
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'dev', url: 'https://github.com/NiravDaraji/infra-gitops.git'
            }
        }

        stage('Init Environment') {
            steps {
                script {
                    env.ENVIRONMENT = params.environment ?: 'dev'
                    env.SELECTED_CHART = params.chartName
                    echo "Using environment: ${env.ENVIRONMENT}"
                    echo "Selected chart: ${env.SELECTED_CHART}"
                }
            }
        }

        stage('Install Missing Tools') {
            steps {
                script {
                    sh '''
                      set -e
                      for t in helm yamllint trivy argocd; do
                        if command -v "$t" >/dev/null 2>&1; then
                          echo "$t found at: $(command -v $t)"
                        else
                          echo "$t not found. Installing..."
                        fi
                      done
                    '''
                }
            }
        }

        /* ============= YAML LINT ============= */
        stage('YAML Lint') {
            steps {
                script {
                    sh """
                        echo 'YAML Lint for chart: ${env.SELECTED_CHART}'
                        
                        # Lint only selected chart templates
                        yamllint -c .yamllint.yaml charts/${env.SELECTED_CHART}/

                        # Lint only selected chart environment values
                        yamllint -c .yamllint.yaml environments/${env.ENVIRONMENT}/values-${env.SELECTED_CHART}.yaml
                    """
                }
            }
        }

        /* ============= HELM LINT ============= */
        stage('Helm Lint') {
            steps {
                script {
                    sh """
                        set -e
                        chartPath="charts/${env.SELECTED_CHART}"
                        valuesFile="environments/${env.ENVIRONMENT}/values-${env.SELECTED_CHART}.yaml"

                        echo "Linting chart: \$chartPath"

                        if [ -f "\$valuesFile" ]; then
                            helm lint "\$chartPath" --values "\$valuesFile"
                        else
                            helm lint "\$chartPath"
                        fi
                    """
                }
            }
        }

        /* ============= HELM UNIT TESTS ============= */
        stage('Helm Unit Tests') {
            steps {
                script {
                    sh """
                        chartPath="charts/${env.SELECTED_CHART}"
                        if [ -d "\$chartPath/tests" ]; then
                            helm unittest "\$chartPath"
                        else
                            echo "No unit tests found. Skipping."
                        fi
                    """
                }
            }
        }

        /* ============= HELM TEMPLATE ============= */
        stage('Helm Template Dry Run') {
            steps {
                script {
                    sh """
                        set -e
                        chartPath="charts/${env.SELECTED_CHART}"
                        valuesFile="environments/${env.ENVIRONMENT}/values-${env.SELECTED_CHART}.yaml"

                        echo "Templating chart: \$chartPath"

                        if [ ! -f "\$valuesFile" ]; then
                            echo "❌ Missing values file: \$valuesFile"
                            exit 1
                        fi

                        helm template "${env.SELECTED_CHART}" "\$chartPath" --values "\$valuesFile"
                    """
                }
            }
        }

        /* ============= TRIVY SCAN ============= */
        stage('Trivy Security Scan') {
            steps {
                script {
                    sh """
                      echo "Running Trivy scan for ${env.SELECTED_CHART}"
                      trivy config charts/${env.SELECTED_CHART} \
                          --severity HIGH,CRITICAL \
                          --include-non-failures \
                          --exit-code 0
                    """
                }
            }
        }

        stage('App Status via ArgoCD') {
            steps {
                sh """
                  argocd login 10.139.9.158:31181 --username admin --password Admin@1234 --insecure
                  argocd app list
                """
            }
        }

        stage('SDLC Summary') {
            steps {
                script {
                    echo "SDLC workflow passed for chart: ${env.SELECTED_CHART}"
                    env.PROMOTION_ELIGIBLE = "true"
                }
            }
        }

        stage('Promotion Approval') {
            when { expression { env.PROMOTION_ELIGIBLE == "true" } }
            steps {
                input message: "Promote ${env.SELECTED_CHART} to STAGING?"
            }
        }

        stage('Trigger Promotion Workflow') {
            when { expression { env.PROMOTION_ELIGIBLE == "true" } }
            steps {
                echo "Triggering promotion for ${env.SELECTED_CHART}"
            }
        }
    }

    post {
        success {
            echo "Pipeline finished successfully."
        }
        failure {
            echo "Pipeline failed."
        }
    }
}
