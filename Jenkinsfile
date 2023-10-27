node {
    try {
        stage('Build') {
            def pythonImage = 'python:2-alpine'
            def sources = ['sources/add2vals.py', 'sources/calc.py']
            
            docker.image(pythonImage).inside {
                sh 'python -m py_compile ' + sources.join(' ')
                stash name: 'compiled-results', includes: 'sources/*.py*'
            }
        }

        stage('Test') {
            def pytestImage = 'qnib/pytest'
            def testArgs = '--verbose --junit-xml test-reports/results.xml sources/test_calc.py'
            
            docker.image(pytestImage).inside {
                sh "py.test $testArgs"
            }

            step([$class: 'JUnitResultArchiver', testResults: 'test-reports/results.xml'])
        }

        stage('Deliver') {
            def volume = "${pwd()}/sources:/src"
            def pyinstallerImage = 'cdrx/pyinstaller-linux:python2'
            def buildId = env.BUILD_ID

            def dockerRunCommand = "pyinstaller -F add2vals.py"

            node("${env.NODE_NAME}") {
                def workspaceDir = pwd()
                def imageExists = docker.image.exists(pyinstallerImage)

                if (!imageExists) {
                    sh "docker pull $pyinstallerImage"
                }

                docker.image(pyinstallerImage).inside("-v $volume") {
                    dir(buildId) {
                        unstash name: 'compiled-results'
                        sh "docker run --rm -v $volume $pyinstallerImage $dockerRunCommand"
                    }
                }

                archiveArtifacts artifacts: "${buildId}/sources/dist/add2vals", onlyIfSuccessful: true

                if (imageExists) {
                    sh "docker run --rm -v $volume $pyinstallerImage 'rm -rf build dist'"
                }
            }
        }
    } catch (Exception e) {
        currentBuild.result = 'FAILURE'
        throw e
    }
}
