pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: rust
    image: rust:1.92-bullseye
    command:
    - cat
    tty: true
    securityContext:
      runAsUser: 0
      runAsGroup: 0
    resources:
      requests:
        memory: '2Gi'
        cpu: '1'
      limits:
        memory: '4Gi'
        cpu: '2'
"""
        }
    }

    parameters {
        string(name: 'GIT_REVISION', defaultValue: '', description: 'Git revision to build (optional)')
        choice(name: 'BUILD_TYPE', choices: ['debug', 'release'], description: 'Build type')
    }

    environment {
        CARGO_TERM_COLOR = 'always'
        RUST_BACKTRACE = '1'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    if (params.GIT_REVISION) {
                        sh "git checkout ${params.GIT_REVISION}"
                    }
                }
            }
        }

        stage('Initialize Submodules') {
            steps {
                sh 'git submodule update --init --recursive'
            }
        }

        stage('Install System Dependencies') {
            steps {
                sh '''
                    # Install system dependencies for Linux builds
                    apt-get update && apt-get install -y build-essential gcc pkg-config libssl-dev
                '''
            }
        }

        stage('Build Shared Library') {
            steps {
                sh '''
                    echo "Building shared library..."
                    cargo build --verbose --package shared --${params.BUILD_TYPE}
                    
                    echo "Running shared library tests..."
                    cargo test --verbose --no-default-features --no-fail-fast --package shared --${params.BUILD_TYPE}
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: "target/${params.BUILD_TYPE}/libshare*,target/${params.BUILD_TYPE}/share*", allowEmptyArchive: true
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

        stage('Build Platform Linux') {
            parallel {
                stage('Linux x86_64') {
                    steps {
                        sh '''
                            # Install x86_64 target
                            rustup target add x86_64-unknown-linux-gnu
                            
                            # Build for x86_64
                            cargo build --verbose --package platform_linux --target x86_64-unknown-linux-gnu --${params.BUILD_TYPE}
                            
                            # Run tests (only for host architecture in this case)
                            if [ "${params.BUILD_TYPE}" = "debug" ]; then
                                cargo test --verbose --package platform_linux --no-default-features --no-fail-fast --target x86_64-unknown-linux-gnu --${params.BUILD_TYPE} || true
                            fi
                        '''
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: 'target/x86_64-unknown-linux-gnu/*/platform_linux', allowEmptyArchive: true
                        }
                    }
                }
            }
        }

        stage('Build WebAssembly') {
            steps {
                sh '''
                    # Install Node.js for WASM tools
                    curl -fsSL https://deb.nodesource.com/setup_18.x | bash -
                    apt-get install -y nodejs
                    
                    # Install WASM-Pack
                    curl https://rustwasm.github.io/wasm-pack/installer/init.sh -sSf | sh
                    
                    # Build WebAssembly
                    cd platform/webassembly/
                    wasm-pack build . --package platform_webassembly --${params.BUILD_TYPE}
                    
                    # Run tests
                    wasm-pack test --node . --package platform_webassembly
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'platform/webassembly/pkg/*', allowEmptyArchive: true
                }
            }
        }
    }
}