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

        APP_BINARY_NAME   = 'add2vals'
        DEPLOYMENT_HOST   = getDeploymentHost()
        DEPLOYMENT_USER   = "${env.DEPLOY_USER}"
        DEPLOYMENT_PORT   = "${env.DEPLOY_PORT}"
        DEPLOYMENT_BRANCH = "${env.DEPLOY_BRANCH}"
        SSH_CRED_ID       = 'host-deploy-key'
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
                                echo "===> Copy binary ke direktori /tmp host server..."
                                scp -P ${DEPLOYMENT_PORT} \
                                    -o StrictHostKeyChecking=no \
                                    dist/${APP_BINARY_NAME} \
                                    ${DEPLOYMENT_USER}@${DEPLOYMENT_HOST}:/tmp/${APP_BINARY_NAME}
                                
                                echo "===> Jalankan binary lifecycle melalui SSH ke server host..."
                                ssh -p ${DEPLOYMENT_PORT} \
                                    -o StrictHostKeyChecking=no \
                                    ${DEPLOYMENT_USER}@${DEPLOYMENT_HOST} << EOF
                                        echo "===> Memulai aplikasi di background."
                                        chmod +x /tmp/${APP_BINARY_NAME}
                                        /tmp/${APP_BINARY_NAME} & APP_PID=\\$!

                                        echo "===> Aplikasi berjalan (PID: \\$APP_PID). Menunggu 1 menit..."
                                        sleep 60

                                        echo "===> Waktu habis. Menghentikan aplikasi."
                                        # Matikan aplikasi menggunakan PID yang disimpan sebelumnya
                                        kill \\$APP_PID || true
                                        rm -f /tmp/${APP_BINARY_NAME}
                                        echo "===> Deployment telah dihapus."
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
