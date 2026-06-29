pipeline {
    agent none

    options {
        skipDefaultCheckout(true)
        timestamps()
    }

    parameters {
        choice(
            name: 'BUILD_TYPE',
            choices: ['DEBUG', 'RELEASE'],
            description: 'Choose Android build type'
        )

        choice(
            name: 'ANDROID_ARTIFACT',
            choices: ['APK', 'AAB'],
            description: 'Choose Android artifact'
        )

        choice(
            name: 'DISTRIBUTION',
            choices: ['NONE', 'FIREBASE', 'STORE'],
            description: 'Choose distribution target'
        )

        string(
            name: 'VERSION_NAME',
            defaultValue: '1.0.0',
            description: 'Android version name'
        )

        string(
            name: 'VERSION_CODE',
            defaultValue: '',
            description: 'Android version code. Empty = Jenkins BUILD_NUMBER'
        )

        text(
            name: 'RELEASE_NOTES',
            defaultValue: 'SmartServe Native Android build',
            description: 'Release notes for Firebase'
        )
    }

    environment {
        ANDROID_HOME = '/Users/enz/Library/Android/sdk'
        ANDROID_SDK_ROOT = '/Users/enz/Library/Android/sdk'

        PATH = "/opt/homebrew/bin:/usr/local/bin:${ANDROID_HOME}/cmdline-tools/latest/bin:${ANDROID_HOME}/platform-tools:/usr/bin:/bin:/usr/sbin:/sbin:${env.PATH}"
    }

    stages {
        stage('Validate Parameters') {
            agent { label 'mac' }

            steps {
                script {
                    if (params.BUILD_TYPE == 'DEBUG' && params.ANDROID_ARTIFACT == 'AAB') {
                        error("DEBUG build only supports APK. Use ANDROID_ARTIFACT=APK.")
                    }

                    if (params.BUILD_TYPE == 'DEBUG' && params.DISTRIBUTION != 'NONE') {
                        error("DEBUG build should use DISTRIBUTION=NONE.")
                    }

                    if (params.DISTRIBUTION == 'FIREBASE' && params.ANDROID_ARTIFACT != 'APK') {
                        error("Firebase App Distribution should use APK.")
                    }

                    if (params.DISTRIBUTION == 'STORE' && params.BUILD_TYPE != 'RELEASE') {
                        error("STORE distribution requires RELEASE build.")
                    }

                    if (params.DISTRIBUTION == 'STORE' && params.ANDROID_ARTIFACT != 'AAB') {
                        error("Play Store upload should use AAB.")
                    }
                }
            }
        }

        stage('Checkout') {
            agent { label 'mac' }

            steps {
                checkout scm
            }
        }

        stage('Prepare Version and Release Notes') {
            agent { label 'mac' }

            steps {
                script {
                    env.APP_VERSION_NAME = params.VERSION_NAME?.trim() ?: "1.0.${env.BUILD_NUMBER}"
                    env.APP_VERSION_CODE = params.VERSION_CODE?.trim() ?: env.BUILD_NUMBER

                    def notes = params.RELEASE_NOTES?.trim()
                    if (!notes) {
                        notes = "SmartServe Native Android build #${env.BUILD_NUMBER}"
                    }

                    writeFile file: 'release-notes.txt', text: notes + "\n"

                    echo "Version name: ${env.APP_VERSION_NAME}"
                    echo "Version code: ${env.APP_VERSION_CODE}"
                    echo "Release notes:"
                    echo readFile('release-notes.txt')
                }

                stash name: 'release-notes', includes: 'release-notes.txt'
            }
        }

        stage('Build Native Android') {
            agent { label 'mac' }

            steps {
                checkout scm
                unstash 'release-notes'

                sh '''
                    echo "Java version:"
                    java -version

                    echo "Gradle version:"
                    ./gradlew --version
                '''

                sh '''
                    rm -rf artifacts
                    rm -rf app/build/outputs/apk
                    rm -rf app/build/outputs/bundle
                    mkdir -p artifacts
                '''

                sh '''
                    bundle config set path vendor/bundle
                    bundle install
                '''

                script {
                    if (params.DISTRIBUTION == 'FIREBASE') {
                        withCredentials([
                            string(credentialsId: 'firebase-android-app-id', variable: 'FIREBASE_ANDROID_APP_ID'),
                            string(credentialsId: 'firebase-token', variable: 'FIREBASE_TOKEN')
                        ]) {
                            sh '''
                                BUILD_TYPE=${BUILD_TYPE} \
                                ANDROID_ARTIFACT=${ANDROID_ARTIFACT} \
                                VERSION_NAME=${APP_VERSION_NAME} \
                                VERSION_CODE=${APP_VERSION_CODE} \
                                RELEASE_NOTES_PATH=release-notes.txt \
                                bundle exec fastlane firebase_release
                            '''
                        }
                    } else if (params.DISTRIBUTION == 'STORE') {
                        sh '''
                            BUILD_TYPE=RELEASE \
                            ANDROID_ARTIFACT=AAB \
                            VERSION_NAME=${APP_VERSION_NAME} \
                            VERSION_CODE=${APP_VERSION_CODE} \
                            bundle exec fastlane build_android

                            echo "Play Store upload lane should run here later."
                        '''
                    } else {
                        sh '''
                            BUILD_TYPE=${BUILD_TYPE} \
                            ANDROID_ARTIFACT=${ANDROID_ARTIFACT} \
                            VERSION_NAME=${APP_VERSION_NAME} \
                            VERSION_CODE=${APP_VERSION_CODE} \
                            bundle exec fastlane build_android
                        '''
                    }
                }

                sh '''
                    find app/build/outputs/apk -name "*.apk" -type f -exec cp {} artifacts/ \\; || true
                    find app/build/outputs/bundle -name "*.aab" -type f -exec cp {} artifacts/ \\; || true

                    echo "Generated artifacts:"
                    ls -lh artifacts || true
                '''

                archiveArtifacts artifacts: 'artifacts/*', allowEmptyArchive: true
            }
        }
    }

    post {
        success {
            echo 'Native Android pipeline completed successfully.'
        }

        failure {
            echo 'Native Android pipeline failed.'
        }
    }
}