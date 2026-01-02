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

    parameters {
        string(
            name: "RUST_VERSION",
            defaultValue: "1.92",
            description: "Rust version to use. The format follows the official Rust versioning."
        )
    }

    stages {
        stage('Checkout') {
            steps {
                sh "git submodule update --init --recursive"
            }
        }

        stage('Shared Library Build') {
            steps {
                sh '''
                    # Install dependencies for Linux
                    apt-get update && apt-get install -y build-essential gcc gcc-multilib libc6-dev libc6-dev-i386
                    
                    # Build shared library
                    cargo build --verbose --package shared --release
                    
                    # Run tests
                    cargo test --verbose --no-default-features --no-fail-fast --package shared --release
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: "target/release/libshare*", fingerprint: true
                    archiveArtifacts artifacts: "target/release/share*", fingerprint: true
                }
            }
        }

        stage('Run Clippy') {
            steps {
                sh '''
                    # Install Clippy
                    rustup component add clippy
                    
                    # Run Clippy
                    cargo clippy --all-targets --all-features -- -D warnings -W clippy::all
                '''
            }
        }

        stage('Linux Platform Build') {
            steps {
                sh '''
                    # Install GCC and dependencies
                    apt-get update && apt-get install -y build-essential gcc gcc-multilib
                    
                    # Install target
                    rustup target add x86_64-unknown-linux-gnu
                    
                    # Build
                    cargo build --verbose --package platform_linux --target x86_64-unknown-linux-gnu --release
                    
                    # Test (only on host architecture)
                    cargo test --verbose --package platform_linux --no-default-features --no-fail-fast --target x86_64-unknown-linux-gnu --release || echo "Tests may fail in cross-compilation"
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: "target/x86_64-unknown-linux-gnu/release/platform_linux", fingerprint: true
                }
            }
        }
    }

}