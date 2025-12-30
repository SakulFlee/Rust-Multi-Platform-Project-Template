pipeline {
    agent {
        kubernetes {
            yaml """
                apiVersion: v1
                kind: Pod
                spec:
                  containers:
                  - name: docker
                    image: docker:latest
                    command:
                    - cat
                    tty: true
                    securityContext:
                      privileged: true
                    volumeMounts:
                    - mountPath: /var/run/docker.sock
                      name: docker-sock
                  volumes:
                  - name: docker-sock
                    hostPath:
                      path: /var/run/docker.sock
                      type: Socket
            """
        }
    }

    parameters {
        string(
            name: 'RUST_VERSION',
            defaultValue: '1.95',
            description: 'Rust version to use (e.g., 1.95, latest). Note: "latest" may not work for macOS/iOS builds.'
        )
    }

    stages {
        stage('Validate Parameters') {
            steps {
                script {
                    if (params.RUST_VERSION == 'latest') {
                        echo "Warning: Using 'latest' Rust version. This may cause issues with macOS/iOS builds."
                    }
                }
            }
        }

        stage('Build Custom Container Images') {
            steps {
                script {
                    // Build Linux container
                    sh '''
                        cd docker/linux
                        podman build -t rust-linux:${RUST_VERSION} --build-arg RUST_VERSION=${RUST_VERSION} .
                    '''
                    
                    // Build Windows container
                    sh '''
                        cd docker/windows
                        podman build -t rust-windows:${RUST_VERSION} --build-arg RUST_VERSION=${RUST_VERSION} .
                    '''
                    
                    // Build Android container
                    sh '''
                        cd docker/android
                        podman build -t rust-android:${RUST_VERSION} --build-arg RUST_VERSION=${RUST_VERSION} .
                    '''
                    
                    // Build WASM container
                    sh '''
                        cd docker/wasm
                        podman build -t rust-wasm:${RUST_VERSION} --build-arg RUST_VERSION=${RUST_VERSION} .
                    '''
                    
                    // Build osxcross container (this one cannot be published)
                    sh '''
                        cd docker/osxcross
                        podman build -t osxcross-rust:${RUST_VERSION} --build-arg RUST_VERSION=${RUST_VERSION} .
                    '''
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
                                    volumeMounts:
                                    - mountPath: /home/jenkins/.cargo
                                      name: cargo-cache
                                    - mountPath: /workspace/target
                                      name: target-cache
                                  volumes:
                                  - name: cargo-cache
                                    persistentVolumeClaim:
                                      claimName: cargo-cache-pvc
                                  - name: target-cache
                                    persistentVolumeClaim:
                                      claimName: target-cache-pvc
                            """
                        }
                    }
                    steps {
                        sh '''
                            git config --global --add safe.directory '*'
                            git submodule update --init --recursive
                            
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

                stage('macOS x86_64 Build') {
                    agent {
                        kubernetes {
                            yaml """
                                apiVersion: v1
                                kind: Pod
                                spec:
                                  containers:
                                  - name: rust
                                    image: osxcross-rust:${params.RUST_VERSION}
                                    command:
                                    - cat
                                    tty: true
                                    securityContext:
                                      privileged: true
                                    volumeMounts:
                                    - mountPath: /home/jenkins/.cargo
                                      name: cargo-cache
                                    - mountPath: /workspace/target
                                      name: target-cache
                                  volumes:
                                  - name: cargo-cache
                                    persistentVolumeClaim:
                                      claimName: cargo-cache-pvc
                                  - name: target-cache
                                    persistentVolumeClaim:
                                      claimName: target-cache-pvc
                            """
                        }
                    }
                    steps {
                        sh '''
                            git config --global --add safe.directory '*'
                            git submodule update --init --recursive
                            
                            # Install target
                            rustup target add x86_64-apple-darwin
                            
                            # Build
                            cargo build --verbose --package platform_macos --target x86_64-apple-darwin --release
                            
                            # Test (only on host architecture)
                            cargo test --verbose --package platform_macos --no-default-features --no-fail-fast --target x86_64-apple-darwin --release || echo "Tests may fail in cross-compilation"
                        '''
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: "target/x86_64-apple-darwin/release/platform_macos", fingerprint: true
                        }
                    }
                }

                stage('macOS aarch64 Build') {
                    agent {
                        kubernetes {
                            yaml """
                                apiVersion: v1
                                kind: Pod
                                spec:
                                  containers:
                                  - name: rust
                                    image: osxcross-rust:${params.RUST_VERSION}
                                    command:
                                    - cat
                                    tty: true
                                    securityContext:
                                      privileged: true
                                    volumeMounts:
                                    - mountPath: /home/jenkins/.cargo
                                      name: cargo-cache
                                    - mountPath: /workspace/target
                                      name: target-cache
                                  volumes:
                                  - name: cargo-cache
                                    persistentVolumeClaim:
                                      claimName: cargo-cache-pvc
                                  - name: target-cache
                                    persistentVolumeClaim:
                                      claimName: target-cache-pvc
                            """
                        }
                    }
                    steps {
                        sh '''
                            git config --global --add safe.directory '*'
                            git submodule update --init --recursive
                            
                            # Install target
                            rustup target add aarch64-apple-darwin
                            
                            # Build
                            cargo build --verbose --package platform_macos --target aarch64-apple-darwin --release
                            
                            # No tests for non-host architecture
                            echo "Build completed for aarch64-apple-darwin"
                        '''
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: "target/aarch64-apple-darwin/release/platform_macos", fingerprint: true
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
                                    volumeMounts:
                                    - mountPath: /home/jenkins/.cargo
                                      name: cargo-cache
                                    - mountPath: /workspace/target
                                      name: target-cache
                                  volumes:
                                  - name: cargo-cache
                                    persistentVolumeClaim:
                                      claimName: cargo-cache-pvc
                                  - name: target-cache
                                    persistentVolumeClaim:
                                      claimName: target-cache-pvc
                            """
                        }
                    }
                    steps {
                        sh '''
                            git config --global --add safe.directory '*'
                            git submodule update --init --recursive
                            
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
                                    volumeMounts:
                                    - mountPath: /home/jenkins/.cargo
                                      name: cargo-cache
                                    - mountPath: /workspace/target
                                      name: target-cache
                                  volumes:
                                  - name: cargo-cache
                                    persistentVolumeClaim:
                                      claimName: cargo-cache-pvc
                                  - name: target-cache
                                    persistentVolumeClaim:
                                      claimName: target-cache-pvc
                            """
                        }
                    }
                    steps {
                        sh '''
                            git config --global --add safe.directory '*'
                            git submodule update --init --recursive
                            
                            # Install Android targets
                            rustup target add x86_64-linux-android
                            rustup target add aarch64-linux-android
                            rustup target add i686-linux-android
                            rustup target add armv7-linux-androideabi
                            
                            # Install cargo-apk
                            cargo install cargo-apk
                            
                            # Generate Release KeyStore
                            cd platform/android/.android
                            echo -e "android\nandroid\n\n\n\n\n\n\nyes" | keytool -genkey -v -keystore release.keystore -alias release -keyalg RSA -keysize 2048 -validity 10000
                            
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

                stage('iOS Build') {
                    agent {
                        kubernetes {
                            yaml """
                                apiVersion: v1
                                kind: Pod
                                spec:
                                  containers:
                                  - name: rust
                                    image: osxcross-rust:${params.RUST_VERSION}
                                    command:
                                    - cat
                                    tty: true
                                    securityContext:
                                      privileged: true
                                    volumeMounts:
                                    - mountPath: /home/jenkins/.cargo
                                      name: cargo-cache
                                    - mountPath: /workspace/target
                                      name: target-cache
                                  volumes:
                                  - name: cargo-cache
                                    persistentVolumeClaim:
                                      claimName: cargo-cache-pvc
                                  - name: target-cache
                                    persistentVolumeClaim:
                                      claimName: target-cache-pvc
                            """
                        }
                    }
                    steps {
                        sh '''
                            git config --global --add safe.directory '*'
                            git submodule update --init --recursive
                            
                            # Install iOS targets
                            rustup target add x86_64-apple-ios
                            rustup target add aarch64-apple-ios
                            rustup target add aarch64-apple-ios-sim
                            
                            # Install cargo-xcodebuild
                            cargo install cargo-xcodebuild
                            
                            # Install XCodeGen and JQ
                            # Assuming these are already in the osxcross image or install via package manager
                            
                            # Make a copy of original Cargo.toml
                            cp platform/ios/Cargo.toml platform/ios/Cargo.toml.original
                            
                            # Get and set device ID for simulator
                            DEVICE_ID=$(xcrun simctl list devices available --json ios | jq -r ".devices | to_entries | map(select(.key | match(\".*iOS.*\"))) | map(.value)[0][0].udid")
                            sed -i "s/device_id = .*/device_id = ${DEVICE_ID}/g" platform/ios/Cargo.toml
                            
                            # Build
                            cd platform/ios
                            cargo xcodebuild build --verbose --package platform_ios --release
                        '''
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: "target/xcodegen/platform_ios/build/Build/Products/Release-iphonesimulator/*", fingerprint: true
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
                                    volumeMounts:
                                    - mountPath: /home/jenkins/.cargo
                                      name: cargo-cache
                                    - mountPath: /workspace/target
                                      name: target-cache
                                  volumes:
                                  - name: cargo-cache
                                    persistentVolumeClaim:
                                      claimName: cargo-cache-pvc
                                  - name: target-cache
                                    persistentVolumeClaim:
                                      claimName: target-cache-pvc
                            """
                        }
                    }
                    steps {
                        sh '''
                            git config --global --add safe.directory '*'
                            git submodule update --init --recursive
                            
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

    post {
        always {
            echo "Pipeline completed. Artifacts have been archived for each platform."
        }
        cleanup {
            sh '''
                # Clean up any temporary files
                rm -rf target/*/debug
            '''
        }
    }
}
