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
                    securityContext:
                      runAsUser: 0
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
                    # Update package lists and install dependencies with sudo in case needed
                    apt-get update && apt-get install -y build-essential gcc gcc-multilib
                    
                    # Verify cargo is available
                    which cargo || echo "Cargo not found"
                    cargo --version || echo "Cargo not working"
                    
                    # Build the Linux platform
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