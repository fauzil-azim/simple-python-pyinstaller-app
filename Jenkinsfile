pipeline {
    agent {
        docker {
            image 'python:3.9-slim'
            args '--user root'
        }
    }

    triggers {
        pollSCM('TZ=Asia/Jakarta \n H/2 * * * *') // Polls every 2 minutes
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

    post {
        always {
            cleanWs() // Clean up the workspace to save disk space on the Jenkins server
        }
    }
}
