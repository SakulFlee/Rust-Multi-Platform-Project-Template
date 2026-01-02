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
                      value: "full"
            """
        }
    }

    stages {
        stage('Checkout') {
            steps {
                sh "git submodule update --init --recursive"
            }
        }

        stage('Setup') {
            steps {
                sh "apt-get update && apt-get install -y build-essential gcc gcc-multilib curl"
            }
        }

        stage('Build') {
            steps {
                sh "cargo build --verbose --package platform_linux --release"
            }
            post {
                always {
                    archiveArtifacts artifacts: "target/release/platform_linux", fingerprint: true
                }
            }
        }
    }
}
