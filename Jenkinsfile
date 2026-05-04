pipeline {
    agent {
        docker {
            image 'python:3.9-slim'
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
                sh 'python -m venv venv'
                sh '. venv/bin/activate'
                sh 'pip install -r requirements.txt'
                sh 'pylint sources/'
                sh 'pytest --junit-xml test-reports/results.xml sources/test_calc.py'
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
