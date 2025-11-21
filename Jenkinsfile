pipeline {
    agent any
    
    environment {
        // Database Configuration
        DB_URL = 'jdbc:oracle:thin:@10.37.0.103:1521/OFSUATDB'
        
        // Java Path
        JAVA_HOME = '/usr/lib/jvm/java-17-openjdk-amd64'
        
        // Maven will be downloaded to workspace
        MAVEN_VERSION = '3.9.9'
        MAVEN_HOME = "${WORKSPACE}/apache-maven-${MAVEN_VERSION}"
        PATH = "${MAVEN_HOME}/bin:${JAVA_HOME}/bin:/usr/local/bin:/usr/bin:/bin"
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code from GitHub...'
                checkout scm
            }
        }
        
        stage('Install Maven') {
            steps {
                echo 'Downloading and installing Maven...'
                sh '''
                    if [ ! -d "${MAVEN_HOME}" ]; then
                        echo "Downloading Maven ${MAVEN_VERSION}..."
                        curl -fsSL https://archive.apache.org/dist/maven/maven-3/${MAVEN_VERSION}/binaries/apache-maven-${MAVEN_VERSION}-bin.tar.gz -o maven.tar.gz
                        tar -xzf maven.tar.gz
                        rm maven.tar.gz
                        echo "Maven installed successfully"
                    else
                        echo "Maven already exists"
                    fi
                '''
            }
        }
        
        stage('Verify Environment') {
            steps {
                echo 'Verifying Java and Maven installation...'
                sh '''
                    echo "JAVA_HOME: ${JAVA_HOME}"
                    echo "MAVEN_HOME: ${MAVEN_HOME}"
                    echo "PATH: ${PATH}"
                    java -version
                    ${MAVEN_HOME}/bin/mvn -version
                '''
            }
        }
        
        stage('Validate') {
            steps {
                echo 'Validating project structure...'
                sh '${MAVEN_HOME}/bin/mvn validate'
            }
        }
        
        stage('Compile') {
            steps {
                echo 'Compiling the project...'
                sh '${MAVEN_HOME}/bin/mvn clean compile'
            }
        }
        
        stage('Clear Liquibase Locks') {
            steps {
                echo 'Clearing any Liquibase locks...'
                withCredentials([usernamePassword(credentialsId: 'HITBL-DB-CREDS',
                                                 usernameVariable: 'DB_USERNAME',
                                                 passwordVariable: 'DB_PASSWORD')]) {
                    sh '''
                        ${MAVEN_HOME}/bin/mvn liquibase:releaseLocks \
                            -Dliquibase.url=${DB_URL} \
                            -Dliquibase.username=${DB_USERNAME} \
                            -Dliquibase.password=${DB_PASSWORD}
                    '''
                }
            }
        }
        
        stage('Fix Checksum Issues') {
            steps {
                echo 'Clearing checksum validation issues...'
                withCredentials([usernamePassword(credentialsId: 'HITBL-DB-CREDS',
                                                 usernameVariable: 'DB_USERNAME',
                                                 passwordVariable: 'DB_PASSWORD')]) {
                    script {
                        try {
                            sh '''
                                ${MAVEN_HOME}/bin/mvn liquibase:clearCheckSums \
                                    -Dliquibase.url=${DB_URL} \
                                    -Dliquibase.username=${DB_USERNAME} \
                                    -Dliquibase.password=${DB_PASSWORD}
                            '''
                            echo 'Checksum cleared successfully'
                        } catch (Exception e) {
                            echo "Checksum clearing failed, but continuing: ${e.getMessage()}"
                        }
                    }
                }
            }
        }
        
        stage('Test Connection') {
            steps {
                echo 'Testing database connection...'
                script {
                    try {
                        withCredentials([usernamePassword(credentialsId: 'HITBL-DB-CREDS',
                                                         usernameVariable: 'DB_USERNAME',
                                                         passwordVariable: 'DB_PASSWORD')]) {
                            sh '''
                                ${MAVEN_HOME}/bin/mvn liquibase:status \
                                    -Dliquibase.url=${DB_URL} \
                                    -Dliquibase.username=${DB_USERNAME} \
                                    -Dliquibase.password=${DB_PASSWORD}
                            '''
                        }
                        echo 'Database connection successful'
                    } catch (Exception e) {
                        echo "Connection test failed, but proceeding with migration: ${e.getMessage()}"
                    }
                }
            }
        }
        
        stage('Liquibase Update') {
            steps {
                echo 'Running Liquibase database migrations...'
                withCredentials([usernamePassword(credentialsId: 'HITBL-DB-CREDS',
                                                 usernameVariable: 'DB_USERNAME',
                                                 passwordVariable: 'DB_PASSWORD')]) {
                    sh '''
                        ${MAVEN_HOME}/bin/mvn liquibase:update \
                            -Dliquibase.url=${DB_URL} \
                            -Dliquibase.username=${DB_USERNAME} \
                            -Dliquibase.password=${DB_PASSWORD}
                    '''
                }
            }
        }
        
        stage('Generate Changelog Report') {
            steps {
                echo 'Generating changelog report...'
                withCredentials([usernamePassword(credentialsId: 'HITBL-DB-CREDS',
                                                 usernameVariable: 'DB_USERNAME',
                                                 passwordVariable: 'DB_PASSWORD')]) {
                    sh '''
                        ${MAVEN_HOME}/bin/mvn liquibase:status \
                            -Dliquibase.url=${DB_URL} \
                            -Dliquibase.username=${DB_USERNAME} \
                            -Dliquibase.password=${DB_PASSWORD} \
                            -Dliquibase.verbose=true
                    '''
                }
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline execution completed'
            archiveArtifacts artifacts: 'target/**/*', fingerprint: true, allowEmptyArchive: true
        }
        success {
            echo 'Database migration completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check the logs for details.'
        }
    }
}
