// Default package version and bug URL
import java.time.*
import java.time.format.DateTimeFormatter
String CURRENTTIME = LocalDateTime.ofInstant(Instant.now(), ZoneOffset.UTC) \
                         .format(DateTimeFormatter.ofPattern("yyyyMMdd"))
String PKGVERS = "corev-openhw-gcc-win64-${CURRENTTIME}"
String BUGURL = 'https://www.embecosm.com'

// This directory is where to expect a MSYS64 installation on the build node
String MSYSHOME = 'C:\\msys64j'

// Bug URL and Package Version override parameters
properties([parameters([
    string(defaultValue: '', description: 'Package Version', name: 'PackageVersion'),
    string(defaultValue: '', description: 'Bug Reporting URL', name: 'BugURL'),
])])

if (params.PackageVersion != '')
  PKGVERS = params.PackageVersion
if (params.BugURL != '')
  BUGURL = params.BugURL
currentBuild.displayName = PKGVERS

node('winbuilder') {
  stage('Cleanup') {
    deleteDir()
  }

  stage('Checkout') {
    checkout scm
    // Store workspace dir in a file we can source later
    bat script: """${MSYSHOME}\\usr\\bin\\cygpath %WORKSPACE% > workspacedir"""

    dir('binutils-gdb') {
      checkout([$class: 'GitSCM',
          branches: [[name: '*/master']],
          extensions: [[$class: 'CloneOption', shallow: true]],
          userRemoteConfigs: [[url: 'https://github.com/pulp-platform/pulp-riscv-binutils-gdb.git']]])
    }
    dir('riscv-gcc') {
      checkout([$class: 'GitSCM',
          branches: [[name: '*/master']],
          extensions: [[$class: 'CloneOption', shallow: true]],
          userRemoteConfigs: [[url: 'https://github.com/pulp-platform/pulp-riscv-gcc.git']]])
    }
    dir('newlib') {
      checkout([$class: 'GitSCM',
          branches: [[name: '*/master']],
          extensions: [[$class: 'CloneOption', shallow: true]],
          userRemoteConfigs: [[url: 'https://github.com/pulp-platform/pulp-riscv-newlib.git']]])
    }
    dir('binutils-gdb-sim') {
      checkout([$class: 'GitSCM',
          branches: [[name: '*/cgen-sim-patch']],
          extensions: [[$class: 'CloneOption', shallow: true]],
          userRemoteConfigs: [[url: 'https://github.com/embecosm/riscv-binutils-gdb.git']]])
    }
    bat script: """set MSYSTEM=MINGW64
                   set /P UNIXWORKSPACE=<workspacedir
                   ${MSYSHOME}\\usr\\bin\\bash --login -c ^
                       "cd %UNIXWORKSPACE% && ./describe-build.sh" """
    archiveArtifacts artifacts: 'build-sources.txt', fingerprint: true
  }

  stage('Build') {
    bat script: """set MSYSTEM=MINGW64
                   set BUGURL=${BUGURL}
                   set PKGVERS=${PKGVERS}
                   set EXTRA_BINUTILS_OPTS=--with-python=no --with-system-readline
                   set /P UNIXWORKSPACE=<workspacedir
                   ${MSYSHOME}\\usr\\bin\\bash --login -c ^
                       "cd %UNIXWORKSPACE% && ./stages/build-riscv32-gcc.sh" """
    bat script: """set MSYSTEM=MINGW64
                   set /P UNIXWORKSPACE=<workspacedir
                   ${MSYSHOME}\\usr\\bin\\bash --login -c ^
                       "cd %UNIXWORKSPACE% && ./utils/extract-mingw-dlls.sh" """
  }

  stage('Package') {
    bat script: """set MSYSTEM=MINGW64
                   set /P UNIXWORKSPACE=<workspacedir
                   ${MSYSHOME}\\usr\\bin\\bash --login -c ^
                       "cd %UNIXWORKSPACE% && utils/prepare-zip-package.sh ${PKGVERS}" """
    archiveArtifacts artifacts: "${PKGVERS}.zip", fingerprint: true
  }

  stage('Test') {
    // Build the CGEN simulator and use it for testing
    bat script: """set MSYSTEM=MINGW64
                   set /P UNIXWORKSPACE=<workspacedir
                   ${MSYSHOME}\\usr\\bin\\bash --login -c ^
                       "cd %UNIXWORKSPACE% && ./stages/test-riscv32-gcc.sh" """
    dir('build/gcc-stage2') {
      archiveArtifacts artifacts: '''gcc/testsuite/gcc/gcc.log,
                                     gcc/testsuite/gcc/gcc.sum,
                                     gcc/testsuite/g++/g++.log,
                                     gcc/testsuite/g++/g++.sum''',
                       fingerprint: true
    }
  }
}
