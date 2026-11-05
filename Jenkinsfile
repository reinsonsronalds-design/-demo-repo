
pipeline {
    agent any

    parameters {
        choice(
            name: 'DEPLOY_TARGET',
            choices: ['current-instance', 'all-instances'],
            description: 'Deployment Target'
        )
        booleanParam(
            name: 'RUN_HEALTH_CHECK',
            defaultValue: true,
            description: 'Run Health Check'
        )
    }

    environment {
        INSTANCE_ID = 'unknown'
        BRANCH_TYPE = 'other'
        DEPLOY_ROOT = '/var/lib/jenkins/ec2-scripts'
    }

    stages {
        stage('Prepare') {
            steps {
                script {
                    env.INSTANCE_ID = sh(
                        script: '''
                            TOKEN=$(curl -s -X PUT \
                              "http://169.254.169.254/latest/api/token" \
                              -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
                            curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
                              http://169.254.169.254/latest/meta-data/instance-id \
                              || echo unknown
                        ''',
                        returnStdout: true
                    ).trim()

                    def branch = env.BRANCH_NAME ?: 'unknown'
                    env.BRANCH_TYPE =
                        branch.startsWith('feature/') ? 'feature' :
                        branch == 'ec2-development' ? 'development' :
                        branch == 'ec2-staging' ? 'staging' :
                        branch == 'main' ? 'production' : 'other'
                }

                echo "Branch: ${env.BRANCH_NAME}"
                echo "Branch Type: ${env.BRANCH_TYPE}"
                echo "Instance ID: ${env.INSTANCE_ID}"
                echo "Deploy Target: ${params.DEPLOY_TARGET}"
            }
        }

        stage('Validate Scripts') {
            steps {
                sh '''
                    echo "=== Script Validation for ${BRANCH_TYPE} ==="
                    found=0
                    for script in *.sh; do
                        if [ -f "$script" ]; then
                            found=1
                            bash -n "$script" || exit 1
                            echo "OK: $script"
                        fi
                    done
                    echo "Script validation completed. Scripts found: $found"
                '''
            }
        }

        stage('Test Scripts') {
            when {
                expression { params.RUN_HEALTH_CHECK == true }
            }
            parallel {
                stage('Syntax Tests') {
                    steps {
                        sh '''
                            if [ -f health-check.sh ]; then
                                bash -n health-check.sh
                                echo "health-check.sh syntax OK"
                            else
                                echo "health-check.sh not found - test skipped"
                            fi
                        '''
                    }
                }

                stage('Functionality Tests') {
                    steps {
                        sh '''
                            if [ -f health-check.sh ]; then
                                chmod +x health-check.sh
                                timeout 30 ./health-check.sh
                            else
                                echo "health-check.sh not found - test skipped"
                            fi
                        '''
                    }
                }
            }
        }

        stage('Deploy to Development') {
            when {
                anyOf {
                    branch 'ec2-development'
                    branch pattern: 'feature/.*', comparator: 'REGEXP'
                }
            }
            steps {
                sh '''
                    mkdir -p "$DEPLOY_ROOT/dev"
                    cp *.sh "$DEPLOY_ROOT/dev/" 2>/dev/null || true
                    echo "Development deployment completed"
                '''
            }
        }

        stage('Deploy to Staging') {
            when {
                branch 'ec2-staging'
            }
            steps {
                sh '''
                    mkdir -p "$DEPLOY_ROOT/staging"
                    cp *.sh "$DEPLOY_ROOT/staging/" 2>/dev/null || true
                    echo "Staging deployment completed"
                '''
            }
        }

        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                input message: 'Deploy to Production Instance?', ok: 'Deploy'
                sh '''
                    mkdir -p "$DEPLOY_ROOT/prod"
                    cp *.sh "$DEPLOY_ROOT/prod/" 2>/dev/null || true
                    chmod +x "$DEPLOY_ROOT"/prod/*.sh 2>/dev/null || true
                    echo "Production deployment completed on ${INSTANCE_ID}"
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: '*.sh', allowEmptyArchive: true
        }
        success {
            echo "Pipeline succeeded on instance ${env.INSTANCE_ID}!"
        }
        failure {
            echo "Pipeline failed on instance ${env.INSTANCE_ID}!"
        }
    }
}
