pipeline {
    agent any
    
    environment {
        GITTOKEN = credentials('gittoken')
    }

    stages {
        stage('Get Code') {
            steps {
                echo 'Downloading code'
                git branch: 'develop', url: 'https://github.com/adrigar94/todo-list-aws.git'
                sh 'ls -la'
                curl -O https://raw.githubusercontent.com/adrigar94/todo-list-aws-config/refs/heads/staging/samconfig.toml
                stash name: 'code', includes: '**/*,.*', useDefaultExcludes: false
                echo WORKSPACE
                sh 'whoami'
                sh 'hostname'
            }
            post {
                always {
                    deleteDir()
                }
            }
        }
        
        stage('Static Test') {
            steps {
                echo WORKSPACE
                sh 'whoami'
                sh 'hostname'
                unstash 'code'
                sh '''
                    export PYTHONPATH=$WORKSPACE
                    flake8 --exit-zero --format=pylint src > static.out
                '''
                recordIssues tools: [flake8(pattern: 'static.out')],
                    qualityGates: [
                        [threshold: 8, type: 'TOTAL', unstable: true],
                        [threshold: 10, type: 'TOTAL', criticality: 'ERROR'],
                    ]
                sh '''
                    bandit -r . -f custom -o security.out --msg-template "{abspath}:{line}: [{test_id}] {msg}" || true
                '''
                recordIssues tools: [pyLint(pattern: 'security.out')],
                    qualityGates: [
                        [threshold: 2, type: 'TOTAL', unstable: true],
                        [threshold: 4, type: 'TOTAL', criticality: 'ERROR'],
                    ]
            }
            post {
                always {
                    deleteDir()
                }
            }
        }
        
        stage('Deploy') {
            steps {
                echo WORKSPACE
                sh 'whoami'
                sh 'hostname'
                unstash 'code'
                sh 'ls -lah'
                sh '''
                    sam build
                      
                    sam deploy \
                      --config-env staging \
                      --config-file samconfig.toml \
                      --no-confirm-changeset \
                      --s3-bucket "" \
                      --resolve-s3 \
                      --no-fail-on-empty-changeset
                      
                    echo "Obteniendo BaseUrlApi desde CloudFormation..."
                    aws cloudformation describe-stacks \
                        --stack-name todo-list-aws-staging \
                        --region us-east-1 \
                        --query "Stacks[0].Outputs[?OutputKey=='BaseUrlApi'].OutputValue" \
                        --output text > baseurl.txt
                '''
                
                script {
                    env.BASE_URL_API = readFile('baseurl.txt').trim()
                    echo "Base URL detectada: ${env.BASE_URL_API}"
                }
            }
            post {
                always {
                    deleteDir()
                }
            }
        }

        stage('Rest Test') {
            steps {
                echo WORKSPACE
                sh 'whoami'
                sh 'hostname'
                echo "${env.BASE_URL_API}"
                unstash 'code'
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    script {
                    def baseUrl = env.BASE_URL_API
                    sh """#!/bin/bash
                        export PYTHONPATH=\$WORKSPACE
                        export BASE_URL="${baseUrl}"
                
                        echo "Esperando a que Flask esté disponible en \$BASE_URL..."
                        for i in \$(seq 1 15); do
                            if curl -s \$BASE_URL/ > /dev/null; then
                                echo "Servicio disponible."
                                break
                            fi
                            echo "Esperando servicio... intento \$i"
                            sleep 1
                            if [ "\$i" -eq 15 ]; then
                                echo "Timeout esperando a los servicios"
                                exit 1
                            fi
                        done
                
                        pytest --junitxml=result-rest.xml test/integration/todoApiTest.py::TestApi
                    """
                }
                }
                junit 'result-rest.xml'
            }
            post {
                always {
                    deleteDir()
                }
            }
        }
        
        

        stage('Promote') {
            steps {
                echo WORKSPACE
                sh 'whoami'
                sh 'hostname'
                unstash 'code'
                sh '''
                    git checkout master
                    git pull origin master
                    git merge origin/develop
                    git push https://$GITTOKEN@github.com/adrigar94/todo-list-aws master
                '''
                
            }
            post {
                always {
                    deleteDir()
                }
            }
        }

    }
}