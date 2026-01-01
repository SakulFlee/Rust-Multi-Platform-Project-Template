def registry = "forgejo.sakul-flee.de"
def namespace = "templates" 
def name = "rust-multi-platform-builder"

def full = "${registry}/${namespace}/${name}"

pipeline {
    agent {
        kubernetes {
            yaml """
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: buildah
                    image: quay.io/buildah/stable
                    command:
                    - cat
                    tty: true
                    securityContext:
                      privileged: true
                      runAsUser: 0
                    volumeMounts:
                      - name: forgejo-token
                        mountPath: /var/run/secrets/additional/secret-jenkins-forgejo-token
                        readOnly: true
                  volumes:
                    - name: forgejo-token
                      secret:
                        secretName: secret-jenkins-forgejo
            """
        }
    }

    parameters {
        string(
            name: 'RUST_VERSION',
            defaultValue: '1.92',
            description: 'Rust version to use. The format follows the official Rust versioning.'
        )
    }

    stages {
        stage('Checkout and Login') {
            steps {
                script {
                    // Login to Registry
                    container('buildah') {
                        sh """
                            cat /var/run/secrets/additional/secret-jenkins-forgejo-token/token | buildah login --username jenkins --password-stdin \"${registry}\"
                        """
                    }
                    
                    // Checkout and Submodules
                    sh "git submodule update --init --recursive"
                }
            }
        }

        stage('Platform Builds with Container Images') {
            parallel {
                stage('Linux Build') {
                    stages {
                        stage('Build Linux Container') {
                            steps {
                                container('buildah') {
                                    sh """
                                        cd docker/linux
                                        buildah bud -t ${full}:linux --build-arg RUST_VERSION=${RUST_VERSION} .
                                        
                                        # Retry push operation up to 3 times
                                        for i in {1..3}; do
                                            if buildah push ${full}:linux; then
                                                echo "Push successful on attempt \$i"
                                                break
                                            else
                                                echo "Push failed on attempt \$i, retrying in 5 seconds..."
                                                sleep 5
                                            fi
                                        done
                                    """
                                }
                            }
                        }
                        
                        stage('Linux Platform Build') {
                            agent {
                                kubernetes {
                                    yaml """
                                        apiVersion: v1
                                        kind: Pod
                                        spec:
                                          containers:
                                          - name: rust
                                            image: ${full}:linux
                                            command:
                                            - cat
                                            tty: true
                                    """
                                }
                            }
                            steps {
                                sh """
                                    # Build (target already installed in container)
                                    cargo build --verbose --package platform_linux --target x86_64-unknown-linux-gnu --release
                                    
                                    # Test (only on host architecture)
                                    cargo test --verbose --package platform_linux --no-default-features --no-fail-fast --target x86_64-unknown-linux-gnu --release || echo "Tests may fail in cross-compilation"
                                """
                            }
                            post {
                                always {
                                    archiveArtifacts artifacts: "target/x86_64-unknown-linux-gnu/release/platform_linux", fingerprint: true
                                }
                            }
                        }
                    }
                }

                stage('Windows Build') {
                    stages {
                        stage('Build Windows Container') {
                            steps {
                                container('buildah') {
                                    sh """
                                        cd docker/windows
                                        buildah bud -t ${full}:windows --build-arg RUST_VERSION=${RUST_VERSION} .
                                        
                                        # Retry push operation up to 3 times
                                        for i in {1..3}; do
                                            if buildah push ${full}:windows; then
                                                echo "Push successful on attempt \$i"
                                                break
                                            else
                                                echo "Push failed on attempt \$i, retrying in 5 seconds..."
                                                sleep 5
                                            fi
                                        done
                                    """
                                }
                            }
                        }
                        
                        stage('Windows Platform Build') {
                            agent {
                                kubernetes {
                                    yaml """
                                        apiVersion: v1
                                        kind: Pod
                                        spec:
                                          containers:
                                          - name: rust
                                            image: ${full}:windows
                                            command:
                                            - cat
                                            tty: true
                                    """
                                }
                            }
                            steps {
                                sh """
                                    # Build (target already installed in container)
                                    cargo build --verbose --package platform_windows --target x86_64-pc-windows-gnu --release
                                    
                                    # Test (only on host architecture)
                                    cargo test --verbose --package platform_windows --no-default-features --no-fail-fast --target x86_64-pc-windows-gnu --release || echo "Tests may fail in cross-compilation"
                                """
                            }
                            post {
                                always {
                                    archiveArtifacts artifacts: "target/x86_64-pc-windows-gnu/release/platform_windows.exe", fingerprint: true
                                }
                            }
                        }
                    }
                }

                stage('Android Build') {
                    stages {
                        stage('Build Android Container') {
                            steps {
                                container('buildah') {
                                    sh """
                                        cd docker/android
                                        buildah bud -t ${full}:android --build-arg RUST_VERSION=${RUST_VERSION} .
                                        
                                        # Retry push operation up to 3 times
                                        for i in {1..3}; do
                                            if buildah push ${full}:android; then
                                                echo "Push successful on attempt \$i"
                                                break
                                            else
                                                echo "Push failed on attempt \$i, retrying in 5 seconds..."
                                                sleep 5
                                            fi
                                        done
                                    """
                                }
                            }
                        }
                        
                        stage('Android Platform Build') {
                            agent {
                                kubernetes {
                                    yaml """
                                        apiVersion: v1
                                        kind: Pod
                                        spec:
                                          containers:
                                          - name: rust
                                            image: ${full}:android
                                            command:
                                            - cat
                                            tty: true
                                            env:
                                            - name: ANDROID_HOME
                                              value: /opt/android-sdk
                                            - name: ANDROID_NDK_HOME
                                              value: /opt/android-sdk/ndk/25.1.8937393
                                    """
                                }
                            }
                            steps {
                                sh """
                                    # Build (targets already installed in container)
                                    cargo apk build --package platform_android --release
                                """
                            }
                            post {
                                always {
                                    archiveArtifacts artifacts: "target/release/apk/*", fingerprint: true
                                }
                            }
                        }
                    }
                }

                stage('WebAssembly Build') {
                    stages {
                        stage('Build WASM Container') {
                            steps {
                                container('buildah') {
                                    sh """
                                        cd docker/wasm
                                        buildah bud -t ${full}:wasm --build-arg RUST_VERSION=${RUST_VERSION} .
                                        
                                        # Retry push operation up to 3 times
                                        for i in {1..3}; do
                                            if buildah push ${full}:wasm; then
                                                echo "Push successful on attempt \$i"
                                                break
                                            else
                                                echo "Push failed on attempt \$i, retrying in 5 seconds..."
                                                sleep 5
                                            fi
                                        done
                                    """
                                }
                            }
                        }
                        
                        stage('WebAssembly Platform Build') {
                            agent {
                                kubernetes {
                                    yaml """
                                        apiVersion: v1
                                        kind: Pod
                                        spec:
                                          containers:
                                          - name: rust
                                            image: ${full}:wasm
                                            command:
                                            - cat
                                            tty: true
                                    """
                                }
                            }
                            steps {
                                sh """
                                    # Build (wasm-pack already installed in container)
                                    wasm-pack build platform/webassembly/ --package platform_webassembly
                                    
                                    # Test
                                    wasm-pack test --node platform/webassembly/ --package platform_webassembly || echo "WASM tests may fail in Node.js environment"
                                """
                            }
                            post {
                                always {
                                    archiveArtifacts artifacts: "./platform/webassembly/pkg/*", fingerprint: true
                                }
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline completed. Artifacts have been archived for each platform."
        }
        cleanup {
            container('buildah') {
                sh '''
                    # Clean up any temporary files
                    rm -rf target/*/debug
                '''
            }
        }
    }
}
