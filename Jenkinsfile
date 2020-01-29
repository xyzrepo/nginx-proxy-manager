pipeline {
	agent {
		label 'docker-multiarch'
	}
	options {
		buildDiscarder(logRotator(numToKeepStr: '5'))
		disableConcurrentBuilds()
	}
	environment {
		IMAGE                      = "nginx-proxy-manager"
		BUILD_VERSION              = getVersion()
		MAJOR_VERSION              = "2"
		COMPOSE_PROJECT_NAME       = "npm_${GIT_BRANCH}_${BUILD_NUMBER}"
		COMPOSE_FILE               = 'docker/docker-compose.ci.yml'
		COMPOSE_INTERACTIVE_NO_CLI = 1
		BUILDX_NAME                = "${COMPOSE_PROJECT_NAME}"
		BRANCH_LOWER               = "${BRANCH_NAME.toLowerCase()}"
	}
	stages {
		stage('Frontend') {
			steps {
				ansiColor('xterm') {
					sh './scripts/frontend-build'
				}
			}
		}
		stage('Backend') {
			steps {
				ansiColor('xterm') {
					sh '''docker build --pull --no-cache --squash --compress \\
						-t "${IMAGE}:ci-${BUILD_NUMBER}" \\
						-f docker/Dockerfile \\
						--build-arg TARGETPLATFORM=linux/amd64 \\
						--build-arg BUILDPLATFORM=linux/amd64 \\
						--build-arg BUILD_VERSION="${BUILD_VERSION}" \\
						--build-arg BUILD_COMMIT="${BUILD_COMMIT}" \\
						--build-arg BUILD_DATE="$(date '+%Y-%m-%d %T %Z')" \\
						.
					'''
				}
			}
		}
		stage('Test') {
			steps {
				ansiColor('xterm') {
					// Bring up a stack
					sh 'docker-compose up -d fullstack'
					sleep 10
					// Run tests
					sh 'rm -rf test/results'
					sh 'docker-compose up --force-recreate cypress'
					// Get results
					sh 'docker cp -L "$(docker-compose ps -q cypress):/results" test/'
				}
			}
			post {
				always {
					junit 'test/results/junit/*'
					// Cypress videos and screenshot artifacts
					dir(path: 'test/results') {
						archiveArtifacts allowEmptyArchive: true, artifacts: '**/*', excludes: '**/*.xml'
					}
					// Dumps to analyze later
					sh 'mkdir -p debug'
					sh 'docker-compose logs fullstack | gzip > debug/docker_fullstack.log.gz'
				}
			}
		}
		stage('Pull Request') {
			when {
				allOf {
					changeRequest()
					not {
						equals expected: 'UNSTABLE', actual: currentBuild.result
					}
				}
			}
			steps {
				ansiColor('xterm') {
					// Dockerhub
					sh 'docker tag ${IMAGE}:ci-${BUILD_NUMBER} docker.io/jc21/${IMAGE}:github-${BRANCH_LOWER}-amd64'
					withCredentials([usernamePassword(credentialsId: 'jc21-dockerhub', passwordVariable: 'dpass', usernameVariable: 'duser')]) {
						sh "docker login -u '${duser}' -p '${dpass}'"
						sh 'docker push docker.io/jc21/${IMAGE}:github-${BRANCH_LOWER}-amd64'
					}

					script {
						def comment = pullRequest.comment("Docker Image for build ${BUILD_NUMBER} is available on [DockerHub](https://cloud.docker.com/repository/docker/jc21/${IMAGE}) as `jc21/${IMAGE}:github-${BRANCH_LOWER}-amd64`")
					}
				}
			}
			post {
				always {
					sh 'docker rmi docker.io/jc21/${IMAGE}:github-${BRANCH_LOWER}-amd64'
				}
			}
		}
		stage('TestBuildX') {
			when {
				allOf {
					branch 'docker-multi'
					not {
						equals expected: 'UNSTABLE', actual: currentBuild.result
					}
				}
			}
			steps {
				ansiColor('xterm') {
					withCredentials([usernamePassword(credentialsId: 'jc21-dockerhub', passwordVariable: 'dpass', usernameVariable: 'duser')]) {
						sh "docker login -u '${duser}' -p '${dpass}'"
						// Buildx to with push
						sh './scripts/buildx --push -t docker.io/jc21/${IMAGE}:docker-multi'
					}
				}
			}
		}
		stage('MultiArch Build') {
			when {
				allOf {
					branch 'master'
					not {
						equals expected: 'UNSTABLE', actual: currentBuild.result
					}
				}
			}
			steps {
				ansiColor('xterm') {
					withCredentials([usernamePassword(credentialsId: 'jc21-dockerhub', passwordVariable: 'dpass', usernameVariable: 'duser')]) {
						sh "docker login -u '${duser}' -p '${dpass}'"
						// Buildx to local files
						sh './scripts/buildx -o type=local,dest=docker-build'
						// Buildx to with push
						sh './scripts/buildx --push -t docker.io/jc21/${IMAGE}:${BUILD_VERSION} -t docker.io/jc21/${IMAGE}:${MAJOR_VERSION}'
					}
				}
			}
		}
	}
	post {
		always {
			sh 'docker-compose down -v --remove-orphans'
			sh 'echo Reverting ownership'
			sh 'docker run --rm -v $(pwd):/data ${DOCKER_CI_TOOLS} chown -R $(id -u):$(id -g) /data'
		}
		success {
			juxtapose event: 'success'
			sh 'figlet "SUCCESS"'
		}
		failure {
			juxtapose event: 'failure'
			sh 'figlet "FAILURE"'
		}
		unstable {
			archiveArtifacts(artifacts: 'debug/**.*', allowEmptyArchive: true)
		}
	}
}

def getVersion() {
	ver = sh(script: 'cat .version', returnStdout: true)
	return ver.trim()
}

def getCommit() {
	ver = sh(script: 'git log -n 1 --format=%h', returnStdout: true)
	return ver.trim()
}
