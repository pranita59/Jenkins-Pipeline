pipeline {
    agent any

    environment {
        PT_REPO = 'https://github.com/ManishaS1713/Jenkins_Pipeline_PT.git'
        JMETER_HOME = 'C:\\Jmeter\\apache-jmeter-5.6.3'
        SLAVE_IP = '192.168.0.147'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()  // Clean previous files
            }
        }

        stage('Checkout Code') {
            steps {
                // Pull latest code from GitHub
                git branch: 'main',
                    url: "${PT_REPO}",
                    credentialsId: 'PT_PipelineToken'
            }
        }

        stage('Verify JMeter') {
            steps {
                bat '''
                REM Set JMeter path and verify installation
                SET JMETER_HOME=%JMETER_HOME%
                %JMETER_HOME%\\bin\\jmeter.bat -v
                '''
            }
        }

        stage('Run Distributed Performance Test') {
            steps {
                bat '''
                REM Set JMeter path
                SET JMETER_HOME=%JMETER_HOME%

                REM Delete old result and report
                IF EXIST performance-result.jtl del performance-result.jtl
                IF EXIST performance-report rmdir /s /q performance-report

                REM Run JMeter distributed test
                %JMETER_HOME%\\bin\\jmeter.bat -n ^
                -t prefScale.jmx ^
                -l performance-result.jtl ^
                -e -o performance-report ^
                -R %SLAVE_IP% ^
                -Jjmeter.save.saveservice.output_format=csv
                '''
            }
        }

        stage('Generate Performance Summary') {
    steps {
        script {
            if (fileExists('performance-result.jtl')) {

                def content = readFile('performance-result.jtl')
                def lines = content.split('\n')

                int total = 0
                int success = 0

                // Start from index 1 to skip header
                for (int i = 1; i < lines.length; i++) {
                    if (lines[i].trim()) {
                        total++
                        if (lines[i].contains(',true,')) {
                            success++
                        }
                    }
                }

                env.TOTAL = total.toString()
                env.SUCCESS = success.toString()

                echo "Total Requests: ${env.TOTAL}"
                echo "Successful Requests: ${env.SUCCESS}"
            }
        }
    }
}
        stage('Publish HTML Report') {
            steps {
                publishHTML(target: [
                    reportDir: 'performance-report',
                    reportFiles: 'index.html',
                    reportName: 'JMeter Performance Report',
                    keepAll: true,
                    alwaysLinkToLastBuild: true,
                    allowMissing: false
                ])
            }
        }

        stage('Email After Performance') {
            steps {
                emailext(
                    subject: "JMeter Test Result - Build #${BUILD_NUMBER}",
                    body: """
                    <h3>Performance Test Summary</h3>
                    <p><b>Total Requests:</b> ${env.TOTAL}</p>
                    <p><b>Successful Requests:</b> ${env.SUCCESS}</p>
                    <p>Check detailed report in Jenkins.</p>
                    """,
                    to: "manishas@ivavsys.com,nikhil@ivavsys.com,rutuja@ivavsys.com,bhavana@ivavsys.com,geeta@ivavsys.com",
                    mimeType: 'text/html'
                )
            }
        }
    }

    post {
        always {
            echo 'Pipeline execution completed.'
        }
        success {
            echo 'Performance test passed successfully!'
        }
        failure {
            echo 'Performance test failed!'
        }
    }
}
