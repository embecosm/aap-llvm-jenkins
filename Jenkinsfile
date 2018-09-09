// Specify parameters to configure which repo and branch to build
properties([parameters([
           string(defaultValue: 'embecosm/aap-llvm', description: 'LLVM Github Repo Name', name: 'LLVMRepoName'),
           string(defaultValue: 'aap-master', description: 'LLVM Branch Name', name: 'LLVMBranchName'),
           string(defaultValue: 'embecosm/aap-clang', description: 'Clang Github Repo Name', name: 'ClangRepoName'),
           string(defaultValue: 'aap-master', description: 'Clang Branch Name', name: 'ClangBranchName'),
           string(defaultValue: 'embecosm/aap-compiler-rt', description: 'Compiler RT Github Repo Name', name: 'CompilerRTRepoName'),
           string(defaultValue: 'aap-master', description: 'Compiler RT Branch Name', name: 'CompilerRTBranchName'),
])])

node ('buildnode') {
  // Cleanup previous build and log files
  stage('Cleanup') {
    sh '''#!/bin/sh
          (cd toolchain && git clean -fxd)
          rm -rf bd-master
          rm -rf install-master
          rm -rf logs-master'''
  }

  // Checkout git repositories
  stage('Checkout') {
      dir('toolchain') {
        git url: 'https://github.com/embecosm/aap-toolchain.git', branch: 'aap-master'
      }
      dir('binutils-gdb') {
        git url: 'https://github.com/embecosm/aap-binutils-gdb', branch: 'aap-master'
      }
      dir('compiler-rt') {
        git url: 'https://github.com/' + '${CompilerRTRepoName}' + '.git', branch: '${CompilerRTBranchName}'
      }
      dir('newlib') {
        git url: 'https://github.com/embecosm/aap-newlib', branch: 'aap-master'
      }
      dir('clang') {
        git url: 'https://github.com/' + '${ClangRepoName}' + '.git', branch: '${ClangBranchName}'
      }
      dir('llvm') {
        git url: 'https://github.com/' + '${LLVMRepoName}' + '.git', branch: '${LLVMBranchName}'
        sh 'cd tools && ln -sf ../../clang'
      }

      // Test components
      dir('gcc') {
        git url: 'https://github.com/embecosm/aap-gcc.git', branch: 'llvm-testing'
      }
  }

  // Build the toolchain
  stage('Build Toolchain') {
    timeout(120) {
      try {
        docker.image('embecosm/buildenv').inside {
          dir('toolchain') {
            sh '''./build-all.sh --no-auto-pull --no-auto-checkout --no-gdbserver --clean'''
          }
        }
      }
      finally {
        archiveArtifacts allowEmptyArchive: true, fingerprint: true, artifacts: 'logs-master/*'
      }
    }
  }
  stage('LLVM regression') {
    timeout(90) {
      try {
        docker.image('embecosm/buildenv').inside {
          dir('bd-master/llvm') {
            sh '''PATH=${WORKSPACE}/install-master/bin:$PATH make check-all 2>&1 | tee "${WORKSPACE}/toolchain/check-llvm.log" || true'''
          }
        }
      }
      finally {
        archiveArtifacts allowEmptyArchive: true, fingerprint: true, artifacts: 'toolchain/check-llvm.log'
      }
    }
  }
  stage('GCC regression') {
    timeout(120) {
      try {
        docker.image('embecosm/buildenv').inside {
          dir('toolchain') {
            sh '''PATH=${WORKSPACE}/install-master/bin:$PATH ./run-tests.py 2>&1 | tee "${WORKSPACE}/toolchain/check-clang-gcc.log" || true'''
          }
        }
      }
      finally {
        archiveArtifacts allowEmptyArchive: true, fingerprint: true, artifacts: 'toolchain/check-clang-gcc.log, toolchain/test-output/gcc.sum, toolchain/test-output/gcc.log'
      }
    }
  }
}
