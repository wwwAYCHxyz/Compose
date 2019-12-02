#!/bin/groovy
def dockerVersions = ''

pipeline {
    agent none

    options {
        skipDefaultCheckout(true)
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timeout(time: 1, unit: 'HOURS')
        parallelsAlwaysFailFast()
    }

    environment {
        TAG = "${env.BUILD_TAG}"
        GOPROXY = "direct"
        DOCKER_BUILDKIT = "1"
    }

    stages {
        stage('Build') {
            parallel {
                stage('Debian') {
                    agent {
                        label 'ubuntu && amd64 && !zfs'
                    }
                    steps {
                        buildImage('debian')
                        checkout scm
                        sh 'docker build -t compose:debian --target build --build-arg BUILD_PLATFORM=debian .'
                        sh 'docker save -o debian.tar compose:debian'
                        stash( includes: "debian.tar", name: "debian" )
                    }
                }
                stage('Alpine') {
                    agent {
                        label 'ubuntu && amd64 && !zfs'
                    }
                    steps {
                        checkout scm
                        sh 'docker build -t compose:alpine --target build --build-arg BUILD_PLATFORM=alpine .'
                        sh 'docker save -o alpine.tar compose:alpine'
                        stash( includes: "alpine.tar", name: "alpine" )
                    }
                }
                stage('Docker') {
                    agent {
                        label 'ubuntu && amd64 && !zfs'
                    }
                    steps {
                        script {
                            dockerVersions = sh(script:"""
                                curl -fs https://api.github.com/repos/docker/docker-ce/releases | jq -r -c '[ .[]
                                | select (.prerelease == false )       # Exclude pre-releases
                                | .tag_name | ltrimstr("v")            # get tag name without v "version" prefix
                                | { major: split(".")[0], tag: . } ]   # group by major version
                                | unique_by(.major)                    # .. to only keep latest
                                | .[].tag'                             # and get matching tag
                            """, returnStdout: true)
                        }
                        echo "${dockerVersions}"
                    }
                }
            }
        }
        stage('Test') {
            steps{
                script {
                    def testMatrix = [:]
                    ['alpine', 'debian'].each { baseImage ->
                        dockerVersions.eachLine { dockerVersion ->
                            ['py27', 'py37'].each { pythonVersion ->
                                testMatrix["${baseImage}_${dockerVersion}_${pythonVersion}"] = runTests(baseImage, dockerVersion, pythonVersion)
                            }
                        }
                    }
                    parallel testMatrix
                }
            }
        }
        stage('Binaries') {
            parallel {
                stage('macosx binary') {
                    agent {
                        label 'mac-python'
                    }
                    steps {
                        checkout scm
                        sh 'script/setup/osx-ci'
                        sh 'tox -e py27,py37 -- tests/unit'
                        sh './script/build/osx'
                        checksum("dist/docker-compose-Darwin-x86_64")
                        archiveArtifacts artifacts: 'dist/*', fingerprint: true
                        dir("dist") {
                            stash name: "bin-darwin"
                        }
                    }
                }
                stage('linux binary') {
                    agent {
                        label 'linux'
                    }
                    steps {
                        checkout scm
                        sh ' ./script/build/linux'
                        checksum("dist/docker-compose-Linux-x86_64")
                        archiveArtifacts artifacts: 'dist/*', fingerprint: true
                        dir("dist") {
                            stash name: "bin-linux"
                        }
                    }
                }
                stage('windows binary') {
                    agent {
                        label 'windows-python'
                    }
                    environment {
                        PATH = "$PATH;C:\\Python37;C:\\Python37\\Scripts"
                    }
                    steps {
                        checkout scm
                        bat 'tox.exe -e py27,py37 -- tests/unit'
                        powershell '.\\script\\build\\windows.ps1'
                        checksum("dist/docker-compose-Windows-x86_64.exe")
                        archiveArtifacts artifacts: 'dist/*', fingerprint: true
                        dir("dist") {
                            stash name: "bin-win"
                        }
                    }
                }
                stage('alpine runtime images') {
                    agent {
                        label 'linux'
                    }
                    steps {
                        buildRuntimeImage('alpine')
                    }
                }
                stage('debian runtime images') {
                    agent {
                        label 'linux'
                    }
                    steps {
                        buildRuntimeImage('debian')
                    }
                }
            }
        }
        stage('Release') {
            when {
                buildingTag()
            }
            parallel {
                stage('Pushing images') {
                    agent {
                        label 'linux'
                    }
                    steps {
                        pushRuntimeImage('alpine')
                        pushRuntimeImage('debian')
                    }
                }
                stage('Creating Github Release') {
                    agent {
                        label 'linux'
                    }
                    steps {
                        checkout scm
                        sh 'mkdir -p dist'
                        dir("dist") {
                            unstash "bin-darwin"
                            unstash "bin-linux"
                            unstash "bin-win"
                            githubRelease("docker/compose")
                        }
                    }
                }
                stage('Publishing Python packages') {
                    agent {
                        label 'linux'
                    }
                    steps {
                        checkout scm
                        withCredentials([[$class: "FileBinding", credentialsId: 'pypirc-docker-dsg-cibot', variable: 'PYPIRC']]) {
                            sh './script/release/python-package'
                        }
                        archiveArtifacts artifacts: 'dist/*', fingerprint: true
                    }
                }
                stage('Publishing binaries to Bintray') {
                    agent {
                        label 'linux'
                    }
                    steps {
                        checkout scm
                        dir("dist") {
                            unstash "bin-darwin"
                            unstash "bin-linux"
                            unstash "bin-win"
                        }
                        withCredentials([usernamePassword(credentialsId: 'bintray-docker-dsg-cibot', usernameVariable: 'BINTRAY_USER', passwordVariable: 'BINTRAY_TOKEN')]) {
                            sh './script/release/push-binaries'
                        }
                    }
                }
            }
        }
    }
}

def buildImage(flavor) {
    checkout scm
    sh "docker build -t compose:${flavor} --target build --build-arg BUILD_PLATFORM=${flavor} ."
    sh "docker save -o ${flavor}.tar compose:${flavor}"
    stash( includes: "${flavor}.tar", name: "${flavor}" )
}

def runTests(baseImage, dockerVersion, pythonVersion) {
    return node( 'ubuntu && amd64 && !zfs' ) {
        stage("${baseImage} ${dockerVersion} ${pythonVersion}") {
           checkout scm
           unstash "${baseImage}"
           sh "docker load -i ${baseImage}.tar"
           sh """
            docker run -t --rm --privileged --volume="\$(pwd)/.git:/code/.git" --volume="/var/run/docker.sock:/var/run/docker.sock" \\
              -e "TAG=compose:${baseImage}" \\
              -e "DOCKER_VERSIONS=${dockerVersion}" \\
              -e "BUILD_NUMBER=\$BUILD_TAG" \\
              -e "PY_TEST_VERSIONS=${pythonVersion}" \\
              --entrypoint="script/test/ci" compose:${baseImage} --verbose
            """
        }
    }
}
