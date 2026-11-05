
pipeline {
    agent any

    parameters {
        choice(name: 'DEPLOY_TARGET',
               choices: ['current-instance', 'all-instances'],
               description: 'Deployment Target')

        booleanParam(name: 'RUN_HEALTH_CHECK',
                     defaultValue: true,
                     description: 'Run Health Check')
    }

    environment {
        INSTANCE_ID = "${sh(script: 'curl -s -m 3 http://169.254.169.254/latest/meta-data/instance-id || echo unknown', returnStdout: true).trim()}"
        BRANCH_TYPE = "${env.BRANCH_NAME.startsWith('feature/') ? 'feature' : env.BRANCH_NAME == 'ec2-development' ? 'development' : env.BRANCH_NAME == 'ec2-staging' ? 'staging' : env.BRANCH_NAME == 'main' ? 'production' : 'other'}"
        DEPLOY_ROOT = '/var/lib/jenkins/ec2-scripts'
    }

    stages {
        stage('Prepare') {
            steps {
                echo "Branch: ${env.BRANCH_NAME}"
                echo "Branch Type: ${env.BRANCH_TYPE}"
                echo "Instance ID: ${INSTANCE_ID}"
                echo "Deploy Target: ${params.DEPLOY_TARGET}"
            }
        }

        stage('Validate Scripts') {
            steps {
                sh '''
                    echo "=== Script Validation for ${BRANCH_TYPE} ==="
                    for script in *.sh; do
                        if [ -f "$script" ]; then
                            bash -n "$script" && echo "OK: $script" || exit 1
                        fi
                    done
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
                        sh 'bash -n health-check.sh && echo "health-check.sh syntax OK"'
                    }
                }
                stage('Functionality Tests') {
                    steps {
                        sh '''
                            chmod +x *.sh
                            timeout 30 ./health-check.sh
                        '''
                    }
                }
            }
        }

        stage('Deploy to Development') {
            when {
                anyOf {
                    branch 'ec2-development'
                    branch 'feature/*'
                }
            }
            steps {
                sh '''
                    mkdir -p $DEPLOY_ROOT/dev
                    cp *.sh $DEPLOY_ROOT/dev/
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
                    mkdir -p $DEPLOY_ROOT/staging
                    cp *.sh $DEPLOY_ROOT/staging/
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
                    mkdir -p $DEPLOY_ROOT/prod
                    cp *.sh $DEPLOY_ROOT/prod/
                    chmod +x $DEPLOY_ROOT/prod/*.sh
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
            echo "Pipeline succeeded on instance ${INSTANCE_ID}!"
        }
        failure {
            echo "Pipeline failed on instance ${INSTANCE_ID}!"
        }
    }
}
