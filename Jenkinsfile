/**
 * CI Stage Pipeline Trigger
 *
 * This is a declarative pipeline for the CI stage pipeline
 * that includes the building of images based on PRs
 *
 */

// Openshift project
openshiftProject = "continuous-infra"
DOCKER_REPO_URL = '172.30.254.79:5000'

// Defaults for SCM operations
env.ghprbGhRepository = env.ghprbGhRepository ?: 'CentOS-PaaS-SIG/ci-pipeline'
env.ghprbActualCommit = env.ghprbActualCommit ?: 'master'

// If this PR does not include an image change, then use this tag
STABLE_LABEL = "stable"
tagMap = [:]

// Initialize
tagMap['jenkins-continuous-infra-slave'] = STABLE_LABEL
tagMap['rpmbuild'] = STABLE_LABEL
tagMap['rsync'] = STABLE_LABEL
tagMap['ostree-compose'] = STABLE_LABEL
tagMap['ostree-image-compose'] = STABLE_LABEL
tagMap['singlehost-test'] = STABLE_LABEL
tagMap['ostree-boot-image'] = STABLE_LABEL
tagMap['linchpin-libvirt'] = STABLE_LABEL

// Fedora Fedmsg Message Provider for stage
MSG_PROVIDER = "fedora-fedmsg-stage"

// IRC properties
IRC_NICK = "ai-coe-bot"
IRC_CHANNEL = "#b4mad"

// CI_MESSAGES known to build successfully
CANNED_CI_MESSAGES = [:]
CANNED_CI_MESSAGES['f26'] = '{"commit":{"username":"zdohnal","stats":{"files":{"README.patches":{"deletions":0,"additions":30,"lines":30},"sources":{"deletions":1,"additions":1,"lines":2},"vim.spec":{"deletions":7,"additions":19,"lines":26},".gitignore":{"deletions":0,"additions":1,"lines":1},"vim-8.0-rhbz1365258.patch":{"deletions":0,"additions":12,"lines":12}},"total":{"deletions":8,"files":5,"additions":63,"lines":71}},"name":"Zdenek Dohnal","rev":"3ff427e02625f810a2cedb754342be44d6161b39","namespace":"rpms","agent":"zdohnal","summary":"Merge branch f25 into f26","repo":"vim","branch":"f26","seen":false,"path":"/srv/git/repositories/rpms/vim.git","message":"Merge branch \'f25\' into f26\n","email":"zdohnal@redhat.com"},"topic":"org.fedoraproject.prod.git.receive"}'
CANNED_CI_MESSAGES['f27'] = '{"commit":{"username":"adrian","stats":{"files":{"criu.spec":{"deletions":0,"additions":5,"lines":5}},"total":{"deletions":0,"files":1,"additions":5,"lines":5}},"name":"Adrian Reber","rev":"386bedee49cb887626140f2c60522751ec620f1d","namespace":"rpms","agent":"adrian","summary":"Adapt ExcludeArch depending on Fedora release","repo":"criu","branch":"f27","seen":false,"path":"/srv/git/repositories/rpms/criu.git","message":"Adapt ExcludeArch depending on Fedora release\\n","email":"adrian@lisas.de"},"topic":"org.fedoraproject.prod.git.receive"}'
// Specific canned messages
CANNED_CI_MESSAGES['f26-singlehost-test'] = '{"commit":{"username":"kdudka","stats":{"files":{"curl.spec":{"deletions":1,"additions":8,"lines":9},"0005-curl-7.53.1-CVE-2017-1000254.patch":{"deletions":0,"additions":136,"lines":136}},"total":{"deletions":1,"files":2,"additions":144,"lines":145}},"name":"Kamil Dudka","rev":"d1d232206aed8ef12596ec3939b72a6476845149","namespace":"rpms","agent":"kdudka","summary":"Resolves: CVE-2017-1000254 - fix out of bounds read in FTP PWD response parser","repo":"curl","branch":"f26","seen":false,"path":"/srv/git/repositories/rpms/curl.git","message":"Resolves: CVE-2017-1000254 - fix out of bounds read in FTP PWD response parser\n","email":"kdudka@redhat.com"},"topic":"org.fedoraproject.prod.git.receive"}'
CANNED_CI_MESSAGES['f27-singlehost-test'] = '{"commit":{"username":"kdudka","stats":{"files":{"curl.spec":{"deletions":4,"additions":11,"lines":15},"0005-curl-7.55.1-CVE-2017-1000254.patch":{"deletions":0,"additions":136,"lines":136}},"total":{"deletions":4,"files":2,"additions":147,"lines":151}},"name":"Kamil Dudka","rev":"9765ef0484ffde44a0104d919799f461b3cb802d","namespace":"rpms","agent":"kdudka","summary":"Resolves: CVE-2017-1000254 - fix out of bounds read in FTP PWD response parser","repo":"curl","branch":"f27","seen":false,"path":"/srv/git/repositories/rpms/curl.git","message":"Resolves: CVE-2017-1000254 - fix out of bounds read in FTP PWD response parser\n","email":"kdudka@redhat.com"},"topic":"org.fedoraproject.prod.git.receive"}'

library identifier: "ci-pipeline@${env.ghprbActualCommit}",
        retriever: modernSCM([$class: 'GitSCMSource',
                              remote: "https://github.com/${env.ghprbGhRepository}",
                              traits: [[$class: 'jenkins.plugins.git.traits.BranchDiscoveryTrait'],
                                       [$class: 'RefSpecsSCMSourceTrait',
                                        templates: [[value: '+refs/heads/*:refs/remotes/@{remote}/*'],
                                                    [value: '+refs/pull/*:refs/remotes/origin/pr/*']]]]])

pipeline {
    agent {
        kubernetes {
            cloud 'openshift'
            label 'stage-trigger-' + env.ghprbActualCommit
            containerTemplate {
                name 'jnlp'
                args '${computer.jnlpmac} ${computer.name}'
                image DOCKER_REPO_URL + '/' + openshiftProject + '/jenkins-continuous-infra-slave:' + STABLE_LABEL
                ttyEnabled false
                command ''
            }
        }
    }
    stages {
        stage("Get Changelog") {
            steps {
                node('master') {
                    script {
                        echo "PR number is: ${env.ghprbPullId}"
                        env.changeLogStr = pipelineUtils.getChangeLogFromCurrentBuild()
                        echo env.changeLogStr
                    }
                    writeFile file: 'changelog.txt', text: env.changeLogStr
                    archiveArtifacts allowEmptyArchive: true, artifacts: 'changelog.txt'
                }
            }
        }
    }
    post {
        always {
            script {
                String prMsg = ""
                if (env.ghprbActualCommit != null && env.ghprbActualCommit != "master") {
                    prMsg = "(PR #${env.ghprbPullId} ${env.ghprbPullAuthorLogin})"
                }
                def message = "${JOB_NAME} ${prMsg} build #${BUILD_NUMBER}: ${currentBuild.currentResult}: ${BUILD_URL}"
                pipelineUtils.sendIRCNotification("${IRC_NICK}-${UUID.randomUUID()}", IRC_CHANNEL, message)
            }
        }
        success {
            echo "yay!"
        }
        failure {
            error "build failed!"
        }
    }
}