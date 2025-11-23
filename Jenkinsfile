pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = '59042090-3267-494b-b18e-adabba7046a3'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
        REACT_APP_VERSION = "1.0.$BUILd_ID"
        
    }

    stages {

        stage('Docker') {
            steps {
                sh 'docker build -t my-playwright .'
            }

        }

        stage('Build') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    ls -la
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la
                '''
            }
        }

        stage('Tests') {
            parallel {
                stage('Unit tests') {
                    agent {
                        docker {
                            image 'node:18-alpine'
                            reuseNode true
                        }
                    }

                    steps {
                        sh '''
                            #test -f build/index.html
                            npm test
                        '''
                    }
                    post {
                        always {
                            junit 'jest-results/junit.xml'
                        }
                    }
                }

                stage('E2E') {
                    agent {
                        docker {
                            image 'my-playwright'
                            reuseNode true
                        }
                    }

                    steps {
                        sh '''
                            serve -s build &
                            sleep 10
                            npx playwright test  --reporter=html
                        '''
                    }

                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Local', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }

        stage('Deploy Staging') {
            agent {
                docker {
                    image 'my-playwright'
                    reuseNode true
                }
            }

            environment {
                 CI_ENVIRONMENT_URL='STAGIN_URL_TO_BE_SET'
            }

            steps {
                sh '''
                    netlify --version
                    
                    echo "🚀 Deploying to Netlify Staging. Site ID: $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --no-build --json > deploy-output.json

                    echo "🔍 Extracting staging deploy URL..."
                    CI_ENVIRONMENT_URL=$(node-jq -r .deploy_url deploy-output.json)

                    echo "🌍 Staging URL: $CI_ENVIRONMENT_URL"
                    echo "🧪 Running Staging E2E Tests..."
                    npx playwright test --reporter=html
                '''
            }

            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'deploy_staging', reportTitles: '', useWrapperFileDirectly: true])
                  
                }
            }
        }

        stage('Deploy prod') {
            agent {
                docker {
                    image 'my-playwright'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL = 'https://regal-concha-c0a784.netlify.app/'
            }

            steps {
                sh '''
                    node --version
                    netlify --version
                    echo "🚀 Deploying to production. Site ID: $NETLIFY_SITE_ID"
                    netlify status
                    netlify deploy --dir=build --prod --no-build
                    npx playwright test  --reporter=html
                '''
            }

            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Deploy_prod', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }
    }
}
