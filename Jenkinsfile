pipeline {
    agent {
        docker {
            image 'python:3.9-slim'
            args '--user root -e HOME=/root'
        }
    }

    triggers {
        githubPush() // Listens for the GitHub Webhook push event
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
        stage('Manual Approval') {
            // Mengecek apakah variabel GIT_BRANCH mengandung kata 'master' pada tipe job Pipeline Standar (bukan Multibranch)
            when { 
                expression { env.GIT_BRANCH == 'master' || env.GIT_BRANCH == 'origin/master' } 
            }
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: "Lanjutkan ke tahap Deploy?"
                }
            }
        }
        stage('Deploy') {
            // This forces Jenkins to exit the global container and execute 
            // the steps below directly on the host server's terminal shell
            agent { label 'built-in' } // Use 'master' in older version of Jenkins
            when { 
                expression { env.GIT_BRANCH == 'master' || env.GIT_BRANCH == 'origin/master' } 
            }
            steps {
                // Creates a clean workspace on the host node and pulls down the stashed code files
                deleteDir()
                unstash 'compiled-results' // Pull the compiled results out of the Jenkins stash
                sh '''
                    echo "===> Compile binary secara langsung di host server..."
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install pyinstaller -r requirements.txt --quiet
                    pyinstaller --onefile sources/add2vals.py

                    echo "===> Copy binary ke direktori /tmp host server..."
                    cp dist/add2vals /tmp/add2vals
                    chmod +x /tmp/add2vals

                    echo "===> Memulai aplikasi di background."
                    # Jalankan binary hasil compile di background dan simpan Process ID (PID)
                    /tmp/add2vals & APP_PID=$!

                    echo "===> Menunggu 1 menit aplikasi berjalan."
                    sleep 60

                    echo "===> Waktu habis. Menghentikan aplikasi."
                    # Matikan aplikasi menggunakan PID yang disimpan sebelumnya
                    kill $APP_PID || true
                    rm -f /tmp/add2vals
                    echo "===> Deployment telah dihapus."
                '''
            }
            post {
                success {
                    archiveArtifacts artifacts: 'dist/add2vals', fingerprint: true
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
