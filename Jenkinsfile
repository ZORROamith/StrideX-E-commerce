pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/ZORROamith/StrideX-E-commerce.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Dependency Security Scan') {
            steps {
                sh '''
                    set +e
                    npm audit --json > npm-audit-report.json
                    AUDIT_STATUS=$?
                    echo "npm audit exit code: $AUDIT_STATUS"
                    npm audit --audit-level=critical
                '''
            }
        }

        stage('SAST - Semgrep') {
            steps {
                sh '''
                    docker run --rm \
                      -v "$WORKSPACE:/src" \
                      -w /src \
                      semgrep/semgrep \
                      semgrep scan --config auto \
                      --json --output semgrep-report.json
                '''
            }
        }

        stage('Lint') {
            steps {
                sh 'npm run lint'
            }
        }

        stage('Production Build') {
            steps {
                sh '''
                    printf "MONGODB_URI=mongodb://127.0.0.1:27017/stridex_build\\nJWT_SECRET=ci_placeholder_secret\\nJWT_EXPIRES_IN=1d\\nRAZORPAY_KEY_ID=build_placeholder\\nRAZORPAY_KEY_SECRET=build_placeholder\\nNEXT_PUBLIC_RAZORPAY_KEY_ID=build_placeholder\\nCLOUDINARY_CLOUD_NAME=build_placeholder\\nCLOUDINARY_API_KEY=build_placeholder\\nCLOUDINARY_API_SECRET=build_placeholder\\n" > .env.local
                    npm run build
                '''
            }
        }

        stage('DAST - OWASP ZAP') {
            steps {
                sh '''
                    set -eu

                    npm run start -- --hostname 0.0.0.0 > app-start.log 2>&1 &
                    echo $! > app.pid

                    READY=0
                    for i in $(seq 1 60); do
                        if curl -fsS http://127.0.0.1:3000/ > /dev/null; then
                            READY=1
                            break
                        fi
                        sleep 2
                    done

                    if [ "$READY" -ne 1 ]; then
                        echo "Application did not become ready. Startup log:"
                        cat app-start.log
                        exit 1
                    fi

                    set +e
                    docker run --rm --network host \
                      -v "$WORKSPACE:/zap/wrk:rw" \
                      ghcr.io/zaproxy/zaproxy:stable \
                      zap-baseline.py \
                      -t http://127.0.0.1:3000 \
                      -m 2 \
                      -r zap-report.html
                    ZAP_STATUS=$?
                    set -e

                    echo "OWASP ZAP exit code: $ZAP_STATUS"

                    if [ "$ZAP_STATUS" -eq 3 ]; then
                        echo "OWASP ZAP encountered a scan error."
                        exit 1
                    fi
                '''
            }
        }
    }

    post {
        always {
            sh '''
                if [ -f app.pid ]; then
                    kill "$(cat app.pid)" 2>/dev/null || true
                fi
            '''
            archiveArtifacts artifacts: 'npm-audit-report.json,semgrep-report.json,zap-report.html,app-start.log', allowEmptyArchive: true
        }

        success {
            echo 'StrideX pipeline completed all configured stages.'
        }

        failure {
            echo 'Pipeline failed. Check the failed stage and Console Output.'
        }
    }
}
