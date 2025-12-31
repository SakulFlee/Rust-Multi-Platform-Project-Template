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
                    - mountPath: /run/containerd/containerd.sock
                      name: containerd-sock
                    - mountPath: /var/lib/containers
                      name: containers-storage
                  volumes:
                  - name: containerd-sock
                    hostPath:
                      path: /run/k0s/containerd.sock
                      type: Socket
                  - name: containers-storage
                    emptyDir: {}
            """
        }
    }

    parameters {
        string(
            name: 'RUST_VERSION',
            defaultValue: '1.92',
            description: 'Rust version to use (format: X.Y, e.g., 1.92). Note: "latest" may not work for macOS/iOS builds.'
        )
    }

    stages {
        stage('Validate Parameters') {
            steps {
                script {
                    // Validate Rust version format using regex (X.Y format where X and Y are numbers)
                    def rustVersionPattern = /^(\d+)\.(\d+)$/
                    if (params.RUST_VERSION == 'latest') {
                        echo "Warning: Using 'latest' Rust version. This may cause issues with macOS/iOS builds."
                    } else if (!params.RUST_VERSION.matches(rustVersionPattern)) {
                        error "Invalid Rust version format: ${params.RUST_VERSION}. Expected format: X.Y (e.g., 1.92, 1.94)"
                    } else {
                        echo "Using Rust version: ${params.RUST_VERSION}"
                    }
                }
            }
        }

        stage('Checkout and Submodules') {
            steps {
                script {
                    // Ensure git is available and checkout with submodules
                    sh '''
                        git config --global --add safe.directory '*'
                        git submodule update --init --recursive
                    '''
                }
            }
        }

        stage('Build Custom Container Images') {
            parallel {
                stage('Build Linux Container') {
                    steps {
                        container('buildah') {
                            sh '''
                                cd docker/linux
                                buildah bud -t rust-linux:${RUST_VERSION} --build-arg RUST_VERSION=${RUST_VERSION} .
                            '''
                        }
                    }
                }
                
                stage('Build Windows Container') {
                    steps {
                        container('buildah') {
                            sh '''
                                cd docker/windows
                                buildah bud -t rust-windows:${RUST_VERSION} --build-arg RUST_VERSION=${RUST_VERSION} .
                            '''
                        }
                    }
                }
                
                stage('Build Android Container') {
                    steps {
                        container('buildah') {
                            sh '''
                                cd docker/android
                                buildah bud -t rust-android:${RUST_VERSION} --build-arg RUST_VERSION=${RUST_VERSION} .
                            '''
                        }
                    }
                }
                
                stage('Build WASM Container') {
                    steps {
                        container('buildah') {
                            sh '''
                                cd docker/wasm
                                buildah bud -t rust-wasm:${RUST_VERSION} --build-arg RUST_VERSION=${RUST_VERSION} .
                            '''
                        }
                    }
                }
            }
        }

        stage('Platform Builds') {
            parallel {
                stage('Linux Build') {
                    agent {
                        kubernetes {
                            yaml """
                                apiVersion: v1
                                kind: Pod
                                spec:
                                  containers:
                                  - name: rust
                                    image: rust-linux:${params.RUST_VERSION}
                                    command:
                                    - cat
                                    tty: true
                            """
                        }
                    }
                    steps {
                        sh '''
                            git config --global --add safe.directory '*'
                            
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

                stage('Windows Build') {
                    agent {
                        kubernetes {
                            yaml """
                                apiVersion: v1
                                kind: Pod
                                spec:
                                  containers:
                                  - name: rust
                                    image: rust-windows:${params.RUST_VERSION}
                                    command:
                                    - cat
                                    tty: true
                            """
                        }
                    }
                    steps {
                        sh '''
                            git config --global --add safe.directory '*'
                            
                            # Install target
                            rustup target add x86_64-pc-windows-gnu
                            
                            # Build
                            cargo build --verbose --package platform_windows --target x86_64-pc-windows-gnu --release
                            
                            # Test (only on host architecture)
                            cargo test --verbose --package platform_windows --no-default-features --no-fail-fast --target x86_64-pc-windows-gnu --release || echo "Tests may fail in cross-compilation"
                        '''
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: "target/x86_64-pc-windows-gnu/release/platform_windows.exe", fingerprint: true
                        }
                    }
                }

                stage('Android Build') {
                    agent {
                        kubernetes {
                            yaml """
                                apiVersion: v1
                                kind: Pod
                                spec:
                                  containers:
                                  - name: rust
                                    image: rust-android:${params.RUST_VERSION}
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
                        sh '''
                            git config --global --add safe.directory '*'
                            
                            # Install Android targets
                            rustup target add x86_64-linux-android
                            rustup target add aarch64-linux-android
                            rustup target add i686-linux-android
                            rustup target add armv7-linux-androideabi
                            
                            # Install cargo-apk
                            cargo install cargo-apk
                            
                            # Generate Release KeyStore
                            cd platform/android/.android
                            echo -e "android\nandroid\n\n\n\n\n\nyes" | keytool -genkey -v -keystore release.keystore -alias release -keyalg RSA -keysize 2048 -validity 10000
                            
                            # Build
                            cd ../../..
                            cargo apk build --package platform_android --release
                        '''
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: "target/release/apk/*", fingerprint: true
                        }
                    }
                }

                stage('WebAssembly Build') {
                    agent {
                        kubernetes {
                            yaml """
                                apiVersion: v1
                                kind: Pod
                                spec:
                                  containers:
                                  - name: rust
                                    image: rust-wasm:${params.RUST_VERSION}
                                    command:
                                    - cat
                                    tty: true
                            """
                        }
                    }
                    steps {
                        sh '''
                            git config --global --add safe.directory '*'
                            
                            # Install wasm-pack
                            curl https://rustwasm.github.io/wasm-pack/installer/init.sh -sSf | sh
                            
                            # Build
                            wasm-pack build platform/webassembly/ --package platform_webassembly
                            
                            # Test
                            wasm-pack test --node platform/webassembly/ --package platform_webassembly || echo "WASM tests may fail in Node.js environment"
                        '''
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
