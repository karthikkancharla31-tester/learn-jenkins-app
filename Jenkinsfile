pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = '59042090-3267-494b-b18e-adabba7046a3'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
    }

    stages {

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
                            image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                            reuseNode true
                        }
                    }

                    steps {
                        sh '''
                            npm install serve
                            node_modules/.bin/serve -s build &
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
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }

            steps {
                sh '''
                    echo "🌐 Installing Netlify CLI & jq..."
                    npm install netlify-cli node-jq

                    echo "🔗 Linking Netlify site..."
                    npx netlify link --id $NETLIFY_SITE_ID

                    echo "🚀 Deploying to Netlify Staging..."
                    npx netlify deploy --dir=build --no-build --json > deploy-output.json

                    echo "🔍 Extracting staging deploy URL..."
                    CI_ENVIRONMENT_URL=$(npx node-jq -r .deploy_url deploy-output.json)

                    echo "🌍 Staging URL: $CI_ENVIRONMENT_URL"

                    echo "🧪 Running Staging E2E Tests..."
                    npx playwright test --reporter=html
                '''
            }

            post {
                always {
                   // publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'deploy_staging', reportTitles: '', useWrapperFileDirectly: true])
                   publishHTML([
                        reportDir: 'playwright-report',
                        reportFiles: 'index.html',
                        reportName: 'Staging E2E'
                    ])
                }
            }
        }

        stage('Approval') {
            steps {
                timeout (time: 15, unit: 'MINUTES') {
                    input message: 'Do you wish to deploy to production', ok: 'Yes, I am sure!'
                }
            }
        }

        stage('Deploy prod') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }

            environment {
                CI_ENVIRONMENT_URL = 'https://regal-concha-c0a784.netlify.app/'
            }

            steps {
                sh '''
                    node --version
                    npm install netlify-cli
                    node_modules/.bin/netlify --version

                    echo "🔗 Linking Netlify Site..."
                    node_modules/.bin/netlify link --id $NETLIFY_SITE_ID

                    echo "🚀 Deploying to production..."
                    node_modules/.bin/netlify deploy --dir=build --prod --no-build
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
