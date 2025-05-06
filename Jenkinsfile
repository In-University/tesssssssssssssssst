pipeline {
    agent any
    environment {
        IMAGE_NAME = "ccpm-restaurant:latest"
        JIRA_URL   = 'https://anhkhoaxn11.atlassian.net'
        JIRA_USER  = '22110355@student.hcmute.edu.vn'
        JIRA_TOKEN = credentials('jira_token')
    }

    stages {
        stage('Clone Repo') {
            steps {
                // Clone repo từ branch hiện tại
                git branch: env.BRANCH_NAME, url: 'https://github.com/In-University/tesssssssssssssssst.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $IMAGE_NAME .'
            }
        }

        stage('Security Check with Trivy') {
            steps {
                sh 'trivy image $IMAGE_NAME > trivy-report.txt'
                archiveArtifacts artifacts: 'trivy-report.txt'
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                    docker stop restaurant-web || true
                    docker rm restaurant-web || true
                    docker run -d --name restaurant-web -p 8085:8085 $IMAGE_NAME
                '''
            }
        }

        stage('Performance Check with Apache Benchmark') {
            steps {
                sh '''
                    echo "Đợi ứng dụng khởi động..."
                    sleep 20
                    ab -n 100 -c 10 http://localhost:8085/ > ab-report.txt
                '''
                archiveArtifacts artifacts: 'ab-report.txt'
            }
        }

        stage('Create Jira Comment') {
            steps {
                script {
                    // 1. Extract issue ID từ commit message
                    def issueId = sh(
                        script: "git log -1 --pretty=%B | grep -oE 'QLNH-[0-9]+' || true",
                        returnStdout: true
                    ).trim()
        
                    if (issueId) {
                        // 2. Build nội dung comment
                        def body = "Kết quả Build:\\n- Trivy Report: trivy-report.txt\\n- Apache Benchmark Report: ab-report.txt"
        
                        // 3. Encode Base64 bằng shell (tránh dùng Groovy getBytes())
                        def authString = sh(
                            script: "echo -n '${JIRA_USER}:${JIRA_TOKEN}' | base64 -w 0",
                            returnStdout: true
                        ).trim()
        
                        // 4. Gọi Jira API tạo comment
                        sh """
                            curl -s -X POST --http1.1 \
                            -H "Authorization: Basic ${authString}" \
                            -H "Content-Type: application/json" \
                            -d '{ "body": "${body}" }' \
                            ${JIRA_URL}/rest/api/2/issue/${issueId}/comment
                        """
                        // 4. Upload Trivy Report if it exists
                        if (fileExists('trivy-report.txt')) {
                            echo "Uploading trivy-report.txt to Jira issue ${issueId}..."
                            sh """
                                curl -s -X POST \
                                -H "Authorization: Basic ${authString}" \
                                -H "X-Atlassian-Token: no-check" \
                                -F "file=@trivy-report.txt" \
                                ${JIRA_URL}/rest/api/2/issue/${issueId}/attachments || echo 'Failed to upload trivy-report.txt, continuing...'
                            """
                        } else {
                            echo "WARN: trivy-report.txt not found. Skipping upload."
                        }

                        // 5. Upload Apache Benchmark Report if it exists
                        if (fileExists('ab-report.txt')) {
                             echo "Uploading ab-report.txt to Jira issue ${issueId}..."
                             sh """
                                curl -s -X POST \
                                -H "Authorization: Basic ${authString}" \
                                -H "X-Atlassian-Token: no-check" \
                                -F "file=@ab-report.txt" \
                                ${JIRA_URL}/rest/api/2/issue/${issueId}/attachments || echo 'Failed to upload ab-report.txt, continuing...'
                             """
                        } else {
                            echo "WARN: ab-report.txt not found. Skipping upload."
                        }
                    } else {
                        echo "No Jira Issue ID found in commit message."
                    }
                }
            }
        }
    }
}
