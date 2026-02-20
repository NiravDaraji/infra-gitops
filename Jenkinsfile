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
            description: 'Select chart to validate '
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
                    
                    def rawEnv = (params.environment ?: '').trim()
                    if (!rawEnv) { rawEnv = 'dev' }
                    env.ENVIRONMENT = rawEnv

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
                              curl -sSL -o /usr/local/bin/argocd \
                                https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
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

        /* ================= YAML LINT ================= */
        stage('YAML Lint') {
            steps {
                script {
                    try {
                        
                        def chartPath  = "charts/${env.SELECTED_CHART}"
                        def valuesPath = "environments/${env.ENVIRONMENT}/values-${env.SELECTED_CHART}.yaml"

                        echo "YAML Lint for chart: ${env.SELECTED_CHART}"

                        echo "Linting templates: ${chartPath}/"
                        sh """
                            set -e
                            yamllint -c .yamllint.yaml "${chartPath}/"
                        """
                        echo "✅ No YAML issues found in ${chartPath}/"

                        if (fileExists(valuesPath)) {
                            echo "Linting values: ${valuesPath}"
                            sh """
                                set -e
                                yamllint -c .yamllint.yaml "${valuesPath}"
                            """
                            echo "✅ No YAML issues found in ${valuesPath}"
                        } else {
                            echo "⚠️ Skipping values lint — file not found: ${valuesPath}"
                        }
                    } catch (err) {
                        error """
❌ SDLC FAILED: YAML LINT ERROR

YAML validation failed for chart: ${env.SELECTED_CHART}.
The stage fails on any yamllint error (syntax/critical issues).
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

Helm template dry-run failed for chart: ${env.SELECTED_CHART}.

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
✔ ArgoCD APP Status Check
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
