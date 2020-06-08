#!/usr/bin/env groovy

pipeline {
    agent {
        dockerfile {
            filename 'docker-agent/AndroidAgent'
        }
    }

    parameters {
        string(name: 'ClientVersion', defaultValue: '', description: 'Client version to build [WARNING: Only add a Version if you have a good reason!]')
        string(name: 'BuildType', defaultValue: '', description: 'Custom BuildType to build [WARNING: Only add a BuildType if you have a good reason!]')
        string(name: 'Flavor', defaultValue: '', description: 'Client version to build [WARNING: Only add a Flavor if you have a good reason!]')

        booleanParam(name: 'AppUnitTests', defaultValue: true, description: 'Run all app unit tests for this build')
        booleanParam(name: 'StorageUnitTests', defaultValue: true, description: 'Run all Storage unit tests for this build')
        booleanParam(name: 'ZMessageUnitTests', defaultValue: false, description: 'Run all zmessaging unit tests for this build')
    }

    stages {
        stage('Prepare Build') {
            steps {
                script {
                    //define the stage
                    last_started = env.STAGE_NAME

                    //define the client_version based on the parameters or env
                    if ("${params.ClientVersion}" != "") {
    					client_version = "${params.ClientVersion}"
    				} else {
     					client_version = env.CLIENT_VERSION
    				}

                    //define the flavor based on the branch
                    if("${params.Flavor}" != "") {
                        flavor = "${params.Flavor}"
                    } else {
                        switch(env.BRANCH_NAME) {
                            case "develop":
                                flavor = "Dev"
                                break
                            case "master":
                                flavor = "Internal"
                                break
                            case "release":
                                flavor = "Prod&Candidate"
                                break
                            default:
                                flavor = "Experimental"
                                break
                        }
                    }

                    //define the default BuildType based on the branch
                    if("${params.BuildType}" != "") {
                        build_type = "${params.BuildType}"
                    } else {
                        switch(env.BRANCH_NAME) {
                            case "release":
                                build_type = "Release"
                                break
                            default:
                                build_type = "Debug"
                                break
                        }
                    }
                }
                configFileProvider([
                        configFile(fileId: '6ed3e6e2-6845-4729-8f8f-cf4c565da6fc', targetLocation: 'app/signing.gradle'),
                        configFile(fileId: 'dc6c5bea-7fff-4dab-a8eb-696b5af3cd6c', targetLocation: 'app/zclient-debug-key.keystore.asc'),
                        configFile(fileId: 'ad99b3ec-cc04-4897-96b0-864151ac38b8', targetLocation: 'app/zclient-release-key.keystore.asc'),
                        configFile(fileId: 'd8c84572-6a63-473b-899c-c160d81b06c9', targetLocation: 'app/zclient-test-key.keystore.asc')
                ]) {}
                sh '''
					base64 --decode app/zclient-debug-key.keystore.asc > app/zclient-debug-key.keystore
                    base64 --decode app/zclient-release-key.keystore.asc > app/zclient-release-key.keystore
                    base64 --decode app/zclient-test-key.keystore.asc > app/zclient-test-key.keystore
				'''
            }
        }

        stage('Precondition Checks') {
            parallel {
                stage('Check SDK/NDK') {
                    steps {
                        script {
                            last_started = env.STAGE_NAME
                        }
                        sh '''echo $ANDROID_HOME
echo $NDK_HOME'''
                    }
                }

                stage('Create local.properties') {
                    steps {
                        sh '''FILE=/local.properties
                                if test -f "$FILE"; then
                                    echo "local.properties exists already"
                                else
                                    echo "sdk.dir="$ANDROID_HOME >> local.properties
                                    echo "ndk.dir="$NDK_HOME >> local.properties
                                fi
                        '''
                    }
                }

                stage('ls') {
                    steps {
                        sh '''ls -la
cd app
ls -la'''
                    }
                }
            }
        }

        stage('App Unit Testing') {
            when {
                expression { params.AppUnitTests }
            }
            steps {
                script {
                    last_started = env.STAGE_NAME
                }
                sh "./gradlew :app:test${flavor}${build_type}UnitTest --parallel"
            }
        }

        stage('Storage Unit Testing') {
            when {
                expression { params.StorageUnitTests }
            }
            steps {
                script {
                    last_started = env.STAGE_NAME
                }
                sh "./gradlew :storage:test${flavor}UnitTest --parallel"
            }
        }

        stage('ZMessage Unit Testing') {
            when {
                expression { params.ZMessageUnitTests }
            }
            steps {
                script {
                    last_started = env.STAGE_NAME
                }
                sh "./gradlew :wire-android-sync-engine:zmessaging:test${build_type}UnitTest -PwireDeflakeTests=1"
            }
        }

        stage('Assemble Client') {
            steps {
                script {
                    last_started = env.STAGE_NAME
                }
                sh "./gradlew --profile assemble${flavor}${build_type} --parallel -x lint"
            }
        }

        stage('Jacoco Report') {
            steps {
                script {
                    last_started = env.STAGE_NAME
                }
                sh './gradlew jacocoTestReport'
                //workaround
                sh 'curl -s https://codecov.io/bash > codecov.sh'
                sh "bash codecov.sh -t ${env.CODECOV_TOKEN}"
            }
        }

        stage('Save artifact') {
            steps {
                script {
                    last_started = env.STAGE_NAME
                }
                archiveArtifacts(artifacts: "app/build/outputs/apk/wire-${flavor.toLowerCase()}-${build_type.toLowerCase()}-$client_version${BUILD_NUMBER}.apk", allowEmptyArchive: true, caseSensitive: true, onlyIfSuccessful: true)
            }
        }

        stage('Upload to S3') {
            steps {
                script {
                    last_started = env.STAGE_NAME
                }
                s3Upload(acl: 'Private', file: "app/build/outputs/apk/wire-${flavor.toLowerCase()}-${build_type.toLowerCase()}-$client_version${BUILD_NUMBER}.apk", bucket: 'z-lohika', path: "megazord/android/${flavor.toLowerCase()}/wire-${flavor.toLowerCase()}-${build_type.toLowerCase()}-$client_version${BUILD_NUMBER}.apk")
            }
        }
    }

    post {
        failure {
            wireSend secret: env.WIRE_BOT_SECRET, message: "${flavor}${build_type} **[${BUILD_NUMBER}](${BUILD_URL})** - ❌ FAILED ($last_started) 👎"
        }
        success {
            script {
                lastCommits = sh(
                        script: "git log -5 --pretty=\"%h [%an] %s\" | sed \"s/^/    /\"",
                        returnStdout: true
                )
            }
            wireSend secret: env.WIRE_BOT_SECRET, message: "${flavor}${build_type} **[${BUILD_NUMBER}](${BUILD_URL})** - ✅ SUCCESS 🎉" +
                    "\nLast 5 commits:\n```\n$lastCommits\n```"
        }
        aborted {
            wireSend secret: env.WIRE_BOT_SECRET, message: "${flavor}${build_type} **[${BUILD_NUMBER}](${BUILD_URL})** - ❌ ABORTED ($last_started) "
        }
    }
}
