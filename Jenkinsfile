def getDeploymentHost() {
    // Initialize the deployment host value with the user's config
    def targetHost = "${DEPLOY_HOST}"

    // Conditionally intercept and perform the dynamic gateway calculation
    if (targetHost == 'localhost' || targetHost == '127.0.0.1') {
        echo "🔍 Host detected as localhost. Dynamically resolving Docker network gateway..."
        
        def resolvedIp = sh(
            script: "docker network inspect bridge --format '{{range .IPAM.Config}}{{if .Gateway}}{{.Gateway}}{{end}}{{end}}'",
            returnStdout: true
        ).trim()
        
        if (resolvedIp) {
            targetHost = resolvedIp
        } else {
            error "❌ Could not derive internal container network gateway IP Address"
        }
    }
    return targetHost
}

pipeline {
    // agent none / any is REQUIRED when mixing built-in and docker agents
    // Any top-level docker agent causes workspace lock conflicts when a stage tries to override with built-in
    agent any

    // In a Jenkins Declarative Pipeline, variables defined inside the global environment {} block 
    // become strictly read-only constants. Jenkins translates the environment {} block into immutable wrapper objects.
    environment {
        PYTHONDONTWRITEBYTECODE       = '1'
        PYTHONUNBUFFERED              = '1'
        PIP_NO_CACHE_DIR              = '1'
        PIP_DISABLE_PIP_VERSION_CHECK = '1'

        APP_BINARY_NAME         = 'add2vals'
        DEPLOYMENT_HOST         = getDeploymentHost()
        DEPLOYMENT_USER         = "${env.DEPLOY_USER}"
        DEPLOYMENT_PORT         = "${env.DEPLOY_PORT}"
        DEPLOYMENT_BRANCH       = "${env.DEPLOY_BRANCH}"
        DEPLOYMENT_INSTALL_DIR  = "/home/${env.DEPLOY_USER}/.local/bin"
        SSH_CRED_ID             = 'host-deploy-key'
    }

    triggers {
        githubPush() // Listens for the GitHub Webhook push event
    }

    stages {
        stage('CI') {
            agent {
                docker {
                    image 'python:3.9-slim-bullseye'
                    args  '--user root -e HOME=/root'
                }
            }
            stages {
                stage('Build') {
                    steps {
                        sh 'python -m py_compile sources/add2vals.py sources/calc.py'
                        stash(name: 'compiled-results', includes: 'sources/*.py*')
                    }
                }
                stage('Test') {
                    steps {
                        // Run everything in one shell so the venv activation persists
                        sh '''
                            python -m venv venv
                            . venv/bin/activate
                            pip install -r requirements.txt

                            # Run Pylint (continue even if score is less than 10)
                            pylint sources/ || true
                            pytest --junit-xml test-reports/results.xml sources/test_calc.py
                        '''
                    }
                    post {
                        always {
                            junit 'test-reports/results.xml' 
                        }
                    }
                }
            }
        }

        stage('Manual Approval') {
            agent none
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: "Lanjutkan ke tahap Deploy?"
                }
            }
        }

        stage('CD') {
            when { 
                expression { 
                    env.GIT_BRANCH == env.DEPLOYMENT_BRANCH || 
                    env.GIT_BRANCH == "origin/${env.DEPLOYMENT_BRANCH}"
                } 
            }
            stages {
                stage('Package') {
                    agent {
                        docker {
                            image 'python:3.9-slim-bullseye'
                            args  '--user root -e HOME=/root'
                        }
                    }
                    steps {
                        unstash 'compiled-results'

                        // Debian 11 (Bullseye) reached its official end-of-life (EOL) - Redirect to the Debian Archive Mirrors. 
                        // Because the release is no longer actively maintained, the Debian security team has moved the package repositories off the main mirrors and archived them. 
                        // When a container runs apt-get update on Debian 11, it pulls an outdated package index, causing apt-get install to point to a file URL that no longer exists.
                        sh '''
                            sed -i 's/deb.debian.org/archive.debian.org/g' /etc/apt/sources.list
                            sed -i 's/security.debian.org/archive.debian.org/g' /etc/apt/sources.list
                            sed -i '/debian-security/d' /etc/apt/sources.list

                            apt-get update -qq
                            apt-get install -y --no-install-recommends binutils
                            . venv/bin/activate
                            pip install pyinstaller
                            pyinstaller --onefile sources/add2vals.py
                        '''
                    }
                    post {
                        success {
                            archiveArtifacts artifacts: 'dist/add2vals', fingerprint: true
                        }
                    }
                }
                stage('Deploy') {
                    // Runs on built-in node (inside jenkins-blueocean)
                    // Has DOCKER_HOST=tcp://docker:2376 from container environment
                    agent { label 'built-in' }
                    steps {
                        sshagent(credentials: [env.SSH_CRED_ID]) {
                            sh '''
                                ssh -p ${DEPLOYMENT_PORT} \
                                    -o StrictHostKeyChecking=no \
                                    ${DEPLOYMENT_USER}@${DEPLOYMENT_HOST} \
                                    "mkdir -p ${DEPLOYMENT_INSTALL_DIR}"

                                echo "===> Uploading binary..."
                                scp -P ${DEPLOYMENT_PORT} \
                                    -o StrictHostKeyChecking=no \
                                    dist/${APP_BINARY_NAME} \
                                    ${DEPLOYMENT_USER}@${DEPLOYMENT_HOST}:${DEPLOYMENT_INSTALL_DIR}/${APP_BINARY_NAME}
                                
                                # Deploy, run, wait, then clean up — all in one remote session
                                # Single session ensures APP_PID is still in scope during cleanup
                                echo "===> Run app lifecycle in a single session..."
                                ssh -p ${DEPLOYMENT_PORT} \
                                    -o StrictHostKeyChecking=no \
                                    ${DEPLOYMENT_USER}@${DEPLOYMENT_HOST} << EOF
                                        APP_BINARY="${DEPLOYMENT_INSTALL_DIR}/${APP_BINARY_NAME}"
                
                                        echo "===> Installing binary..."
                                        chmod +x "\\${APP_BINARY}"

                                        echo "===> Verifying binary is accessible from PATH..."
                                        which "${APP_BINARY_NAME}" && echo "PATH check OK: \\$(which ${APP_BINARY_NAME})"

                                        echo "===> Running in background..."
                                        "\\${APP_BINARY}" & APP_PID=\\$! 
                                        echo "App started — PID: \\${APP_PID}"

                                        echo "===> Testing app..."
                                        echo "9 + 8 = \\$(\\${APP_BINARY} 9 8)"                                        
                                        echo "19 + 81 = \\$(\\${APP_BINARY} 19 81)"
                                        echo "919 + 181 = \\$(\\${APP_BINARY} 919 181)"

                                        echo "===> Waiting 60s..."
                                        sleep 60

                                        echo "===> Stopping app..."
                                        if kill -0 \\${APP_PID} 2>/dev/null; then
                                            kill \\${APP_PID}
                                            wait \\${APP_PID} 2>/dev/null || true
                                            echo "App stopped cleanly"
                                        else
                                            echo "App already exited"
                                        fi

                                        echo "===> Removing binary..."
                                        rm -f "\\${APP_BINARY}"

                                        echo "===> Cleanup complete"
                            '''
                        }
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Deployed build #${env.BUILD_NUMBER} - ${env.GIT_COMMIT}"
        }
        failure {
            echo "Pipeline FAILED on ${env.GIT_BRANCH} — ${env.GIT_COMMIT}"
        }
        always {
            cleanWs() // Clean up the workspace to save disk space on the Jenkins server
        }
    }
}
