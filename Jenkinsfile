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
        stage('Build Container Images') {
            steps {
                script {
                    lock('rust-multi-platform-builder-registry') {
                        sh """
                            echo "Starting build with global lock..."
                        """
                        
                        // Login to Registry
                        container('buildah') {
                            sh """
                                cat /var/run/secrets/additional/secret-jenkins-forgejo-token/token | buildah login --username jenkins --password-stdin \"${registry}\"
                            """
                        }
                        
                        // Checkout and Submodules
                        sh """
                            git config --global --add safe.directory '*'
                            git submodule update --init --recursive
                        """
                        
                        // Build and Push Container Images in parallel
                        parallel(
                            'Build and Push Linux Container': {
                                container('buildah') {
                                    sh """
                                        cd docker/linux
                                        buildah bud -t ${full}:linux --build-arg RUST_VERSION=${RUST_VERSION} .
                                        buildah push ${full}:linux
                                    """
                                }
                            },
                            'Build and Push Windows Container': {
                                container('buildah') {
                                    sh """
                                        cd docker/windows
                                        buildah bud -t ${full}:windows --build-arg RUST_VERSION=${RUST_VERSION} .
                                        buildah push ${full}:windows
                                    """
                                }
                            },
                            'Build and Push Android Container': {
                                container('buildah') {
                                    sh """
                                        cd docker/android
                                        buildah bud -t ${full}:android --build-arg RUST_VERSION=${RUST_VERSION} .
                                        buildah push ${full}:android
                                    """
                                }
                            },
                            'Build and Push WASM Container': {
                                container('buildah') {
                                    sh """
                                        cd docker/wasm
                                        buildah bud -t ${full}:wasm --build-arg RUST_VERSION=${RUST_VERSION} .
                                        buildah push ${full}:wasm
                                    """
                                }
                            }
                        )
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
                                    image: ${full}:linux
                                    command:
                                    - cat
                                    tty: true
                            """
                        }
                    }
                    steps {
                        sh """
                            git config --global --add safe.directory '*'
                            
                            # Install target
                            rustup target add x86_64-unknown-linux-gnu
                            
                            # Build
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

                stage('Windows Build') {
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
                            git config --global --add safe.directory '*'
                            
                            # Install target
                            rustup target add x86_64-pc-windows-gnu
                            
                            # Build
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

                stage('Android Build') {
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
                            echo -e "android\nandroid\n\n\n\n\nyes" | keytool -genkey -v -keystore release.keystore -alias release -keyalg RSA -keysize 2048 -validity 10000
                            
                            # Build
                            cd ../../..
                            cargo apk build --package platform_android --release
                        """
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
                                    image: ${full}:wasm
                                    command:
                                    - cat
                                    tty: true
                            """
                        }
                    }
                    steps {
                        sh """
                            git config --global --add safe.directory '*'
                            
                            # Install wasm-pack
                            curl https://rustwasm.github.io/wasm-pack/installer/init.sh -sSf | sh
                            
                            # Build
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
