pipeline {
    agent {
        kubernetes {
            defaultContainer 'rust' 
            yaml """
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: rust
                    image: rust:latest
                    imagePullPolicy: always 
                    command: ["cat"]
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
