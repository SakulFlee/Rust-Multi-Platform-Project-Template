pipeline {
    agent {
        kubernetes {
            yaml """
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: rust
                    image: rust:1.92
                    command:
                    - cat
                    tty: true
                    env:
                    - name: CARGO_TERM_COLOR
                      value: "always"
                    - name: RUST_BACKTRACE
                      value: "1"
            """
        }
    }

    stages {
        stage('Checkout') {
            steps {
                sh "git submodule update --init --recursive"
            }
        }

        stage('Build') {
            steps {
                sh '''
                    # Update package lists and install dependencies
                    apt-get update && apt-get install -y build-essential gcc gcc-multilib
                    
                    # Build the Linux platform (this will also build dependencies including shared)
                    cargo build --verbose --package platform_linux --release
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: "target/release/platform_linux", fingerprint: true
                }
            }
        }
    }
}