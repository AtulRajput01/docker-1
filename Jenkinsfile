pipeline {
    agent any

    environment {
        IMAGE_NAME = "spar-back"
        NEW_STAGE_TAG = "latest"
        CONTAINER_PORT = "3000"
        HOST_PORT = "3000"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def buildResult = sh(
                        script: '''#!/bin/bash
set -eo pipefail
echo "Building Docker image..."
docker build -t ${IMAGE_NAME}:latest . | tee failure.log
''',
                        returnStatus: true
                    )

                    if (buildResult != 0) {
                        error "❌ Docker build failed! Check failure.log"
                    }
                }
            }
        }

        stage('Run New Docker Container') {
            steps {
                script {
                    sh '''#!/bin/bash
echo "Starting new container with latest image..."
docker run -d -p ${HOST_PORT}:${CONTAINER_PORT} ${IMAGE_NAME}
'''
                }
            }
        }

        stage('ZAP Security Scan') {
            steps {
                script {
                    sh '''#!/bin/bash
echo "⚡ Starting ZAP Security Scan..."

TARGET_URL="http://localhost:${HOST_PORT}"

# Start the ZAP scan
SCAN_ID=$(curl -s "http://localhost:8089/JSON/ascan/action/scan/?url=${TARGET_URL}" | jq -r '.scan')

# If SCAN_ID is null, something failed
if [ "$SCAN_ID" = "null" ] || [ -z "$SCAN_ID" ]; then
    echo "❌ Failed to start ZAP scan. Exiting."
    exit 1
fi

echo "🛰️ Scan started with ID: $SCAN_ID"

# Wait until the scan completes
while true; do
    STATUS=$(curl -s "http://localhost:8089/JSON/ascan/view/status/?scanId=$SCAN_ID" | jq -r '.status')
    echo "📊 Scan progress: $STATUS%"
    if [ "$STATUS" = "100" ]; then
        break
    fi
    sleep 5
done

echo "✅ Scan completed. Saving report..."

# Save the HTML report
curl "http://localhost:8089/OTHER/core/other/htmlreport/" -o zap_report.html
'''
                }
                archiveArtifacts artifacts: 'zap_report.html', onlyIfSuccessful: true
            }
        }
    }
}
