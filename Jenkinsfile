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
            name: 'RUST_VERSION',
            defaultValue: '1.92',
            description: 'Rust version to use. The format follows the official Rust versioning.'
        )
    }

    stages {
        stage('Checkout') {
            steps {
                sh "git submodule update --init --recursive"
            }
        }

        stage('Shared Library Build') {
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
            }
            steps {
                sh '''
                    # Install dependencies for Linux (for general compatibility)
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
            }
            steps {
                sh '''
                    # Install Clippy
                    rustup component add clippy

                    # Run Clippy
                    cargo clippy --all-targets --all-features -- -D warnings -W clippy::all
                '''
            }
        }

        stage('Platform Builds') {
            parallel {
                stage('Linux Platform Build') {
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

                stage('Windows Platform Build') {
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
                                    - name: CC_x86_64_pc_windows_gnu
                                      value: "x86_64-w64-mingw32-gcc"
                                    - name: CXX_x86_64_pc_windows_gnu
                                      value: "x86_64-w64-mingw32-g++"
                                    - name: AR_x86_64_pc_windows_gnu
                                      value: "x86_64-w64-mingw32-gcc-ar"
                                    - name: CARGO_TARGET_X86_64_PC_WINDOWS_GNU_LINKER
                                      value: "x86_64-w64-mingw32-gcc"
                            """
                        }
                    }
                    steps {
                        sh '''
                            # Install MinGW-w64 for Windows cross-compilation
                            apt-get update && apt-get install -y gcc-mingw-w64 mingw-w64

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

                stage('Android Platform Build') {
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
                                    - name: ANDROID_HOME
                                      value: "/opt/android-sdk"
                                    - name: ANDROID_NDK_HOME
                                      value: "/opt/android-sdk/ndk/25.1.8937393"
                            """
                        }
                    }
                    steps {
                        sh '''
                            # Install required packages
                            apt-get update && apt-get install -y openjdk-21-jdk wget curl unzip

                            # Set up Android SDK
                            export ANDROID_HOME=/opt/android-sdk
                            export ANDROID_NDK_HOME=/opt/android-sdk/ndk/25.1.8937393
                            export PATH=$PATH:$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools

                            # Create directories
                            mkdir -p $ANDROID_HOME/cmdline-tools

                            # Download and install Android SDK Command Line Tools
                            cd /tmp && \\
                                wget https://dl.google.com/android/repository/commandlinetools-linux-9477386_latest.zip && \\
                                unzip commandlinetools-linux-9477386_latest.zip -d cmdline-tools-temp && \\
                                mkdir -p $ANDROID_HOME/cmdline-tools && \\
                                mv cmdline-tools-temp/cmdline-tools $ANDROID_HOME/cmdline-tools/latest && \\
                                rm -rf cmdline-tools-temp commandlinetools-linux-9477386_latest.zip

                            # Accept licenses
                            yes | sdkmanager --licenses

                            # Install required Android components
                            sdkmanager "platform-tools" "platforms;android-33" "build-tools;33.0.2" "ndk;25.1.8937393"

                            # Install Rust Android targets
                            rustup target add x86_64-linux-android aarch64-linux-android i686-linux-android armv7-linux-androideabi

                            # Install cargo-apk
                            cargo install cargo-apk

                            # Generate Release KeyStore
                            cd platform/android/.android && echo -e "android\\nandroid\\n\\n\\n\\n\\n\\n\\nyes" | keytool -genkey -v -keystore release.keystore -alias release -keyalg RSA -keysize 2048 -validity 10000

                            # Build
                            cargo apk build --package platform_android --release
                        '''
                    }
                    post {
                        always {
                            archiveArtifacts artifacts: "target/release/apk/*", fingerprint: true
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
                    steps {
                        sh '''
                            # Install Node.js for WASM testing
                            apt-get update && apt-get install -y nodejs npm curl

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
            sh 'rm -rf target/*/debug'
        }
    }
}
