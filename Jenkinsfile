pipeline {
    agent any

    environment {
        IMAGE_NAME = "spar-back"
        NEW_STAGE_TAG = "latest"
        CONTAINER_PORT = "3000"
        HOST_PORT = "3000"
        TARGET_URL = "http://localhost:3000"
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
echo "🛠️ Building Docker image..."
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
echo "🚀 Starting container from latest image..."
docker run -d -p ${HOST_PORT}:${CONTAINER_PORT} ${IMAGE_NAME}
sleep 10  # Give container time to boot up
'''
                }
            }
        }

        stage('ZAP Security Scan') {
            steps {
                script {
                    sh '''#!/bin/bash
set -e

echo "⚡ Starting ZAP Security Scan..."

# Spider first to populate the ZAP site tree
SPIDER_ID=$(curl -s "http://localhost:8089/JSON/spider/action/scan/?url=${TARGET_URL}" | jq -r '.scan')

if [ "$SPIDER_ID" = "null" ] || [ -z "$SPIDER_ID" ]; then
    echo "❌ Failed to start ZAP spider. Exiting."
    exit 1
fi

echo "🕷️ Spider started with ID: $SPIDER_ID"

# Wait for spider to finish
while true; do
    STATUS=$(curl -s "http://localhost:8089/JSON/spider/view/status/?scanId=$SPIDER_ID" | jq -r '.status')
    echo "📊 Spider progress: $STATUS%"
    [ "$STATUS" = "100" ] && break
    sleep 5
done

# Start active scan now that site is known to ZAP
SCAN_ID=$(curl -s "http://localhost:8089/JSON/ascan/action/scan/?url=${TARGET_URL}" | jq -r '.scan')

if [ "$SCAN_ID" = "null" ] || [ -z "$SCAN_ID" ]; then
    echo "❌ Failed to start active scan. Exiting."
    exit 1
fi

echo "🔍 Active scan started with ID: $SCAN_ID"

# Wait for scan to finish
while true; do
    STATUS=$(curl -s "http://localhost:8089/JSON/ascan/view/status/?scanId=$SCAN_ID" | jq -r '.status')
    echo "📊 Scan progress: $STATUS%"
    [ "$STATUS" = "100" ] && break
    sleep 5
done

# Save report
echo "📄 Generating ZAP HTML report..."
curl -s "http://localhost:8089/OTHER/core/other/htmlreport/" -o zap_report.html
'''
                }
                archiveArtifacts artifacts: 'zap_report.html', onlyIfSuccessful: true
            }
        }
    }
}
