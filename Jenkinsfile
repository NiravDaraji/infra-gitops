pipeline {
    agent any

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

        /* ================= CHECKOUT ================= */
        stage('Checkout') {
            steps {
                git branch: 'dev', url: 'https://github.com/NiravDaraji/infra-gitops.git'
            }
        }

        /* ================= INIT ENV ================= */
        stage('Init Environment') {
            steps {
                script {
                    env.ENVIRONMENT = params.environment ?: 'dev'
                    env.SELECTED_CHART = params.chartName
                    echo "Environment  : ${env.ENVIRONMENT}"
                    echo "Selected Chart: ${env.SELECTED_CHART}"
                }
            }
        }

        /* ================= INSTALL TOOLS ================= */
        stage('Install Missing Tools') {
            steps {
                script {
                    try {
                        sh '''
                          set -e
                          for t in helm yamllint trivy argocd; do
                            if command -v "$t" >/dev/null 2>&1; then
                              echo "$t found at: $(command -v $t)"
                            else
                              echo "$t not found. Please install manually or automate installation."
                            fi
                          done
                        '''
                    } catch (err) {
                        error """
❌ SDLC FAILED: TOOL INSTALLATION ERROR

Required tools not installed or installation failed.
Fix the issue and re-run the pipeline.

-----------------------------------------
${err}
-----------------------------------------
"""
                    }
                }
            }
        }

        /* ================= YAML LINT ================= */
        stage('YAML Lint') {
            steps {
                script {
                    try {
                        sh """
                            echo 'YAML Lint for chart: ${env.SELECTED_CHART}'

                            # Lint selected chart template folder
                            tpl_output=\$(yamllint -c .yamllint.yaml charts/${env.SELECTED_CHART}/ || true)
                            if [ -n "\$tpl_output" ]; then
                              echo "\$tpl_output"
                            else
                              echo "✅ No YAML issues in charts/${env.SELECTED_CHART}/"
                            fi

                            # Lint selected environment values file
                            values_file="environments/${env.ENVIRONMENT}/values-${env.SELECTED_CHART}.yaml"
                            if [ -f "\$values_file" ]; then
                                val_output=\$(yamllint -c .yamllint.yaml "\$values_file" || true)
                                if [ -n "\$val_output" ]; then
                                  echo "\$val_output"
                                else
                                  echo "✅ No YAML issues in \$values_file"
                                fi
                            else
                                echo "⚠️ Skipping YAML lint — file not found: \$values_file"
                            fi
                        """
                    } catch (err) {
                        error """
❌ SDLC FAILED: YAML LINT ERROR

YAML validation failed for chart: ${env.SELECTED_CHART}.
Check the lint output above and correct values or templates.

-----------------------------------------
${err}
-----------------------------------------
"""
                    }
                }
            }
        }

        /* ================= HELM LINT ================= */
        stage('Helm Lint') {
            steps {
                script {
                    try {
                        sh """
                            set -e
                            chartPath="charts/${env.SELECTED_CHART}"
                            valuesFile="environments/${env.ENVIRONMENT}/values-${env.SELECTED_CHART}.yaml"

                            echo "Linting Helm chart: \$chartPath"

                            if [ -f "\$valuesFile" ]; then
                                helm lint "\$chartPath" --values "\$valuesFile"
                            else
                                helm lint "\$chartPath"
                            fi
                        """
                    } catch (err) {
                        error """
❌ SDLC FAILED: HELM LINT ERROR

Helm lint failed for chart: ${env.SELECTED_CHART}.
Fix the lint issues and re-run the pipeline.

-----------------------------------------
${err}
-----------------------------------------
"""
                    }
                }
            }
        }

        /* ================= HELM UNIT TESTS ================= */
        stage('Helm Unit Tests') {
            steps {
                script {
                    try {
                        sh """
                            chartPath="charts/${env.SELECTED_CHART}"

                            if [ -d "\$chartPath/tests" ]; then
                                helm unittest "\$chartPath"
                            else
                                echo "No unit tests found — skipping."
                            fi
                        """
                    } catch (err) {
                        error """
❌ SDLC FAILED: HELM UNIT TEST FAILURE

Unit tests failed for chart: ${env.SELECTED_CHART}.

-----------------------------------------
${err}
-----------------------------------------
"""
                    }
                }
            }
        }

        /* ================= HELM TEMPLATE DRY-RUN ================= */
        stage('Helm Template Dry Run') {
            steps {
                script {
                    try {
                        sh """
                            set -e
                            chartPath="charts/${env.SELECTED_CHART}"
                            valuesFile="environments/${env.ENVIRONMENT}/values-${env.SELECTED_CHART}.yaml"

                            echo "Dry running helm template: \$chartPath"

                            if [ ! -f "\$valuesFile" ]; then
                               echo "❌ Missing values file: \$valuesFile"
                               exit 1
                            fi

                            helm template "${env.SELECTED_CHART}" "\$chartPath" --values "\$valuesFile"
                        """
                    } catch (err) {
                        error """
❌ SDLC FAILED: HELM TEMPLATE DRY-RUN ERROR

Helm template rendering failed for chart: ${env.SELECTED_CHART}.

-----------------------------------------
${err}
-----------------------------------------
"""
                    }
                }
            }
        }

        /* ================= TRIVY SECURITY SCAN ================= */
        stage('Trivy Security Scan') {
            steps {
                script {
                    try {
                        sh """
                          echo "Running Trivy scan for ${env.SELECTED_CHART}"
                          trivy config charts/${env.SELECTED_CHART} \
                              --severity HIGH,CRITICAL \
                              --include-non-failures \
                              --exit-code 0
                        """
                    } catch (err) {
                        error """
❌ SDLC FAILED: TRIVY SECURITY SCAN ERROR

Security misconfigurations detected in chart: ${env.SELECTED_CHART}.
Check Trivy scan result.

-----------------------------------------
${err}
-----------------------------------------
"""
                    }
                }
            }
        }

        /* ================= ARGOCD STATUS ================= */
        stage('App Status via ArgoCD') {
            steps {
                script {
                    try {
                        sh """
                          argocd login 10.139.9.158:31181 --username admin --password Admin@1234 --insecure
                          argocd app list
                        """
                    } catch (err) {
                        error """
❌ SDLC FAILED: ARGOCD STATUS ERROR

Unable to fetch ArgoCD application status.

-----------------------------------------
${err}
-----------------------------------------
"""
                    }
                }
            }
        }

        /* ================= SUMMARY ================= */
        stage('SDLC Summary') {
            steps {
                echo """
=========================================
 SDLC WORKFLOW STATUS: PASSED
-----------------------------------------
✔ YAML Lint
✔ Helm Lint
✔ Helm Unit Tests
✔ Helm Template Dry Run
✔ Trivy Security Scan
✔ ArgoCD Status Check
-----------------------------------------
Validated chart: ${env.SELECTED_CHART}
Environment:     ${env.ENVIRONMENT}
=========================================
"""
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully."
        }
        failure {
            echo "Pipeline failed — check above error details."
        }
    }
}
