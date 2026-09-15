pipeline {
    // agent none is REQUIRED when mixing built-in and docker agents
    // Any top-level docker agent causes workspace lock conflicts
    // when a stage tries to override with built-in
    agent none

    environment {
        PYTHONDONTWRITEBYTECODE       = '1'
        PYTHONUNBUFFERED              = '1'
        PIP_NO_CACHE_DIR              = '1'
        PIP_DISABLE_PIP_VERSION_CHECK = '1'
        APP_BINARY_NAME   = 'add2vals'
        DEPLOYMENT_HOST   = "${env.DEPLOY_HOST}"
        DEPLOYMENT_USER   = "${env.DEPLOY_USER}"
        DEPLOYMENT_PORT   = "${env.DEPLOY_PORT}"
        DEPLOYMENT_BRANCH = 'local-master'
        SSH_CRED_ID       = 'host-deploy-key'
    }

    triggers {
        pollSCM('TZ=Asia/Jakarta \n H/2 * * * *') // Polls every 2 minutes
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

        stage('CD') {
            // Mengecek apakah variabel GIT_BRANCH mengandung kata 'local-master' pada tipe job Pipeline Standar (bukan Multibranch)
            when { 
                expression { 
                    env.GIT_BRANCH == env.DEPLOYMENT_BRANCH || 
                    env.GIT_BRANCH == "origin/${env.DEPLOYMENT_BRANCH}"
                } 
            }
            stages {
                stage('Manual Approval') {
                    agent none
                    steps {
                        timeout(time: 30, unit: 'MINUTES') {
                            input message: "Lanjutkan ke tahap Deploy?"
                        }
                    }
                }
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
                        archiveArtifacts artifacts: 'dist/add2vals', fingerprint: true
                    }
                }
                stage('Deploy') {
                    // Runs on built-in node (inside jenkins-blueocean)
                    // Has DOCKER_HOST=tcp://docker:2376 from compose environment,
                    // so docker network inspect works against the DinD daemon
                    agent { label 'built-in' }
                    steps {
                        script {
                            // Resolve Gateway
                            env.DEPLOYMENT_HOST = sh(
                                script: '''
                                docker network inspect bridge --format '{{range .IPAM.Config}}{{if .Gateway}}{{.Gateway}}{{end}}{{end}}'
                                ''',
                                returnStdout: true
                            ).trim()

                            if (!env.DEPLOYMENT_HOST) {
                                error "Could not derive gateway from container IP"
                            }

                            echo "===> Container IP gateway: ${env.DEPLOYMENT_HOST}"
                        }

                        sshagent(credentials: [env.SSH_CRED_ID]) {
                            sh '''
                                echo "===> Copy binary ke direktori /tmp host server..."
                                scp -v -P ${DEPLOYMENT_PORT} \
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
            // cleanWs requires an explicit node context when top-level agent is none
            node('built-in') {
                cleanWs() // Clean up the workspace to save disk space on the Jenkins server
            }
        }
    }
}
