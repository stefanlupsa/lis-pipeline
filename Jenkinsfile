#!/usr/bin/env groovy

def RunPowershellCommand(psCmd) {
    bat "powershell.exe -NonInteractive -ExecutionPolicy Bypass -Command \"[Console]::OutputEncoding=[System.Text.Encoding]::UTF8;$psCmd;EXIT \$global:LastExitCode\""
}

def GetTestResults(type) {
    withEnv(["Type=${type}"]) {
        def returnValues = powershell returnStdout: true, script: '''$file = Get-Item ".\\Report\\*-junit.xml" | Sort-Object LastWriteTimeUtc | select -Last 1
           $content = [xml](Get-Content $file)
           $failCase = [int]($content.testsuites.testsuite.failures)
           $allCase = [int]($content.testsuites.testsuite.tests)
           $abortCase = [int]($content.testsuites.testsuite.errors)
           $skippedCase = [int]($content.testsuites.testsuite.skipped)
           $passCase = $allCase - $failCase - $abortCase - $skippedCase

           if ($env:Type -eq "pass") {
               return $passCase
           } elseif ($env:Type -eq "abort") {
               return $abortCase
           } elseif ($env:Type -eq "fail") {
               return $failCase
           } elseif ($env:Type -eq "skipped") {
               return $skippedCase
           } elseif ($env:Type -eq "all") {
               return $allCase
           }
           '''
        return "${returnValues}"
    }
}

def reportStageStatus(stageName, stageStatus) {
    script {
        env.STAGE_NAME_REPORT = stageName
        env.STAGE_STATUS_REPORT = stageStatus
    }
    withCredentials(bindings: [file(credentialsId: 'KERNEL_QUALITY_REPORTING_DB_CONFIG',
                                    variable: 'PERF_DB_CONFIG')]) {
        dir('kernel_version_report' + env.BUILD_NUMBER + env.BRANCH_NAME) {
              unstash 'kernel_version_ini'
              sh '''#!/bin/bash
                  bash "${WORKSPACE}/scripts/reporting/report_stage_state.sh" \
                      --pipeline_name "pipeline-msft-kernel-validation/${BRANCH_NAME}" \
                      --pipeline_build_number "${BUILD_NUMBER}" \
                      --pipeline_stage_status "${STAGE_STATUS_REPORT}" \
                      --pipeline_stage_name "${STAGE_NAME_REPORT}" \
                      --kernel_info "./scripts/package_building/kernel_versions.ini" \
                      --kernel_source "MSFT" --kernel_branch "${KERNEL_GIT_BRANCH}" \
                      --distro_version "${DISTRO_VERSION}" --db_config ${PERF_DB_CONFIG} || true
              '''
        }
    }
}

def getVhdLocation(basePath, distroVersion) {
    def distroFamily = distroVersion.split('_')[0]
    return "${basePath}\\" + distroFamily + "\\" + distroVersion + "\\" + distroVersion + ".vhdx"
}

def getTests(priority, platform) {
    def AZURE_VALIDATION_TESTS_HASH=[:]
    def HYPERV_VALIDATION_TESTS_HASH=[:]

    AZURE_VALIDATION_TESTS_HASH_P0 = [AZURE_P0_VALIDATION:"-TestPriority 0 -ExcludeTests 'NVIDIA-CUDA-DRIVER-VALIDATION,NVIDIA-GRID-DRIVER-VALIDATION,VERIFY-DPDK-COMPLIANCE'"]
    HYPERV_VALIDATION_TESTS_HASH_P0 = [HV_P0_VALIDATION:"-TestPriority 0 -ExcludeTests 'SRIOV-VERIFY-LSPCI,SRIOV-VERIFY-SINGLE-VF-CONNECTION,LIVE-MIGRATE,LIVE-MIGRATE-QUICK-MIGRATE,NVME-DISK-VALIDATION,NVME-FSTRIM,NVME-BLKDISCARD'"]
    HYPERV_VALIDATION_SRIOV_TESTS_HASH_P0 = [HV_P0_SRIOV_VALIDATION:"-TestPriority 0 -TestCategory Functional -TestArea SRIOV"]
    HYPERV_VALIDATION_NVME_TESTS_HASH_P0 = [HV_P0_NVME_VALIDATION:"-TestPriority 0 -TestCategory Functional -TestArea NVME"]
    HYPERV_VALIDATION_MIGRATE_TESTS_HASH_P0 = [HV_P0_MIGRATE_VALIDATION:"-TestPriority 0 -TestCategory Functional -TestArea MIGRATION"]

    AZURE_VALIDATION_TESTS_HASH_P1 = [AZURE_P1_FUNCTIONAL_VALIDATION:"-TestPriority 1 -TestCategory Functional -ExcludeTests 'NVIDIA-CUDA-DRIVER-VALIDATION-MAX-GPU,NVIDIA-GRID-DRIVER-VALIDATION-MAX-GPU,INFINIBAND-OPEN-MPI-2VM,VERIFY-DPDK-FAILSAFE-DURING-TRAFFIC,VERIFY-DPDK-RING-LATENCY,GPU-PCI-RESCIND'"]
    HYPERV_VALIDATION_TESTS_HASH_P1 = [HV_P1_FUNCTIONAL_VALIDATION:"-TestCategory Functional -TestPriority 1 -ExcludeTests 'LIVE-MIGRATE-FAILOVER,DYNAMIC-MEMORY-HIGH-PRIORITY,SRIOV-DISABLE-ENABLE-PCI,SRIOV-VERIFY-SINGLE-VF-CONNECTION-ONE-VCPU,SRIOV-VERIFY-SINGLE-VF-CONNECTION-MAX-VCPU,SRIOV-VERIFY-MAX-VF-CONNECTION,SRIOV-VERIFY-MAX-VF-CONNECTION-MAX-VCPU,SRIOV-ADD-MAX-NICS-DURING-PROVISION,SRIOV-INTERRUPTS-CHANGE,SRIOV-DISABLEVF-ON-GUEST,SRIOV-RELOAD-MODULE,SRIOV-DISABLEVF-PING,SRIOV-DISABLEVF-IPERF,SRIOV-DETACH-NIC,SRIOV-DISABLEVF-ONHOST,SRIOV-DISABLE-NIC,SRIOV-DISABLE-VMQ,SRIOV-CHANGE-RSS,SRIOV-MEASURE-VF-FAILBACK,SRIOV-MULTICAST,SRIOV-BROADCAST,SRIOV-REBOOT-VM,SRIOV-STRESS-SAVE,SRIOV-STRESS-PAUSE,NVME-DISK-OPERATIONS,NVME-CHECK-EXPECTED-FAILURES,NVME-PCI-RESCIND'",
        HV_P1_STRESS_VALIDATION:"-TestPriority 1 -TestCategory Stress"]
    HYPERV_VALIDATION_SRIOV_TESTS_HASH_P1 = [HV_P1_SRIOV_VALIDATION:"-TestPriority '0,1' -TestCategory Functional -TestArea SRIOV"]
    HYPERV_VALIDATION_NVME_TESTS_HASH_P1 = [HV_P1_NVME_VALIDATION:"-TestPriority '0,1' -TestCategory Functional -TestArea NVME"]
    HYPERV_VALIDATION_MIGRATE_TESTS_HASH_P1 = [HV_P1_MIGRATE_VALIDATION:"-TestPriority '0,1' -TestCategory Functional -TestArea MIGRATION"]

    AZURE_VALIDATION_TESTS_HASH_P2 = [AZURE_P2_FUNCTIONAL_VALIDATION:"-TestPriority 2 -TestCategory Functional",
        AZURE_P2_COMMUNITY_VALIDATION:"-TestPriority 2 -TestCategory Community",
        AZURE_P2_STRESS_VALIDATION:"-TestPriority 2 -TestCategory Stress"]
    HYPERV_VALIDATION_TESTS_HASH_P2= [HV_P2_FUNCTIONAL_VALIDATION:"-TestCategory Functional -TestPriority 2 -ExcludeTests 'SRIOV-IPERF-STRESS,SRIOV-MOVE-VHD'",
        HV_P2_STRESS_VALIDATION:"-TestPriority 2 -TestCategory Stress",
        HV_P2_COMMUNITY_VALIDATION:"-TestPriority 2 -TestCategory Community"]
    HYPERV_VALIDATION_SRIOV_TESTS_HASH_P2 = [HV_P2_SRIOV_VALIDATION:"-TestPriority '0,1,2' -TestCategory Functional -TestArea SRIOV"]

    AZURE_VALIDATION_TESTS_HASH_P3 = [AZURE_P3_COMMUNITY_VALIDATION:"-TestPriority 3 -TestCategory Community"]
    HYPERV_VALIDATION_TESTS_HASH_P3= [HV_P3_VALIDATION:"-TestPriority 3"]

    if ("${priority}" == "P0") {
        AZURE_VALIDATION_TESTS_HASH += AZURE_VALIDATION_TESTS_HASH_P0
        HYPERV_VALIDATION_TESTS_HASH += HYPERV_VALIDATION_TESTS_HASH_P0
    }
    if ("${priority}" == "P1") {
        AZURE_VALIDATION_TESTS_HASH += AZURE_VALIDATION_TESTS_HASH_P0
        HYPERV_VALIDATION_TESTS_HASH += HYPERV_VALIDATION_TESTS_HASH_P0
        AZURE_VALIDATION_TESTS_HASH += AZURE_VALIDATION_TESTS_HASH_P1
        HYPERV_VALIDATION_TESTS_HASH += HYPERV_VALIDATION_TESTS_HASH_P1
    }
    if ("${priority}" == "P2") {
        AZURE_VALIDATION_TESTS_HASH += AZURE_VALIDATION_TESTS_HASH_P0
        HYPERV_VALIDATION_TESTS_HASH += HYPERV_VALIDATION_TESTS_HASH_P0
        AZURE_VALIDATION_TESTS_HASH += AZURE_VALIDATION_TESTS_HASH_P1
        HYPERV_VALIDATION_TESTS_HASH += HYPERV_VALIDATION_TESTS_HASH_P1
        AZURE_VALIDATION_TESTS_HASH += AZURE_VALIDATION_TESTS_HASH_P2
        HYPERV_VALIDATION_TESTS_HASH += HYPERV_VALIDATION_TESTS_HASH_P2
    }
    if ("${priority}" == "P3") {
        AZURE_VALIDATION_TESTS_HASH += AZURE_VALIDATION_TESTS_HASH_P0
        HYPERV_VALIDATION_TESTS_HASH += HYPERV_VALIDATION_TESTS_HASH_P0
        AZURE_VALIDATION_TESTS_HASH += AZURE_VALIDATION_TESTS_HASH_P1
        HYPERV_VALIDATION_TESTS_HASH += HYPERV_VALIDATION_TESTS_HASH_P1
        AZURE_VALIDATION_TESTS_HASH += AZURE_VALIDATION_TESTS_HASH_P2
        HYPERV_VALIDATION_TESTS_HASH += HYPERV_VALIDATION_TESTS_HASH_P2
        AZURE_VALIDATION_TESTS_HASH += AZURE_VALIDATION_TESTS_HASH_P3
        HYPERV_VALIDATION_TESTS_HASH += HYPERV_VALIDATION_TESTS_HASH_P3
    }
    if ("${priority}" == "P0-SRIOV") {
        HYPERV_VALIDATION_TESTS_HASH += HYPERV_VALIDATION_SRIOV_TESTS_HASH_P0
    }
    if ("${priority}" == "P1-SRIOV") {
        HYPERV_VALIDATION_TESTS_HASH += HYPERV_VALIDATION_SRIOV_TESTS_HASH_P1
    }
    if ("${priority}" == "P2-SRIOV") {
        HYPERV_VALIDATION_TESTS_HASH += HYPERV_VALIDATION_SRIOV_TESTS_HASH_P2
    }
    if ("${priority}" == "P0-NVME") {
        HYPERV_VALIDATION_TESTS_HASH += HYPERV_VALIDATION_NVME_TESTS_HASH_P0
    }
    if ("${priority}" == "P1-NVME") {
        HYPERV_VALIDATION_TESTS_HASH += HYPERV_VALIDATION_NVME_TESTS_HASH_P1
    }
    if ("${priority}" == "P0-MIGRATE") {
        HYPERV_VALIDATION_TESTS_HASH += HYPERV_VALIDATION_MIGRATE_TESTS_HASH_P0
    }
    if ("${priority}" == "P1-MIGRATE") {
        HYPERV_VALIDATION_TESTS_HASH += HYPERV_VALIDATION_MIGRATE_TESTS_HASH_P1
    }
    if ("${platform}".contains("Azure")) {
        return AZURE_VALIDATION_TESTS_HASH
    }
    if ("${platform}".contains("HyperV")) {
        return HYPERV_VALIDATION_TESTS_HASH
    }
}

def prepareEnv(branch, remote, distroVersion) {
    cleanWs()
    git branch: branch, url: remote
    script {
      env.AZURE_OS_IMAGE = env.AZURE_UBUNTU_IMAGE_BIONIC
      env.PACKAGE_TYPE = "deb"
      if (distroVersion.toLowerCase().contains("centos")) {
        env.AZURE_OS_IMAGE = env.AZURE_CENTOS_7_IMAGE
        env.PACKAGE_TYPE = "rpm"
      }
    }
}

def unstashKernel(kernelStash) {
    unstash kernelStash
    powershell """
        \$rmPath = "\${env:ProgramFiles}\\Git\\usr\\bin\\rm.exe"
        \$basePath = "./scripts/package_building/${env.BUILD_NUMBER}-${env.BRANCH_NAME}-${kernelStash}/*/${env.PACKAGE_TYPE}"

        & \$rmPath -rf "\${basePath}/*dbg*"
        & \$rmPath -rf "\${basePath}/*devel*"
        & \$rmPath -rf "\${basePath}/*debug*"
    """
}


pipeline {
  parameters {
    string(defaultValue: "stable", description: 'Branch to be built', name: 'KERNEL_GIT_BRANCH')
    string(defaultValue: "stable", description: 'Branch label (stable or unstable)', name: 'KERNEL_GIT_BRANCH_LABEL')
    choice(choices: 'Ubuntu_18.04.1\nCentOS_7.5', description: 'Distro version.', name: 'DISTRO_VERSION')
    choice(choices: 'False\nTrue', description: 'Enable kernel debug', name: 'KERNEL_DEBUG')
    choice(choices: 'P0\nP1\nP2\nP3\n', description: 'Priority', name: 'PRIORITY')
    string(defaultValue: "build_artifacts, publish_temp_artifacts, boot_test, publish_artifacts, publish_azure_vhd, validation, validation_hyperv, validation_sriov_hyperv, validation_nvme_hyperv, validation_migrate_hyperv, validation_azure, publish_results",
           description: 'What stages to run', name: 'ENABLED_STAGES')
  }
  environment {
    LISAV2_REMOTE = "https://github.com/lis/LISAv2.git"
    LISAV2_BRANCH = "master"
    LISAV2_AZURE_REGION = "westus2"
    LISAV2_RG_IDENTIFIER = "msftk"
    LISAV2_AZURE_VM_SIZE_SMALL = "Standard_A2"
    LISAV2_AZURE_VM_SIZE_LARGE = "Standard_E64_v3"
    KERNEL_ARTIFACTS_PATH = 'kernel-artifacts'
    BUILD_PATH = '/mnt/tmp/kernel-build-folder'
    KERNEL_CONFIG = 'Microsoft/config-azure'
    CLEAN_ENV = 'True'
    USE_CCACHE = 'True'
    BUILD_NAME = 'm'
    FOLDER_PREFIX = 'msft'
    AZURE_UBUNTU_IMAGE_BIONIC = "Canonical UbuntuServer 18.04-DAILY-LTS latest"
    AZURE_CENTOS_7_IMAGE = "OpenLogic CentOS 7.5 latest"
    FUNC_FAIL_ONAZURE = 0
    FUNC_FAIL_ONLOCAL = 0
    FUNC_PASS_ONAZURE = 0
    FUNC_PASS_ONLOCAL = 0
    FUNC_ABORT_ONAZURE = 0
    FUNC_ABORT_ONLOCAL = 0
    FUNC_SKIP_ONAZURE = 0
    FUNC_SKIP_ONLOCAL = 0
  }
  options {
    overrideIndexTriggers(false)
  }
  agent {
    node {
      label 'meta_slave'
    }
  }
  stages {
          stage('build_artifacts_ubuntu') {
              when {
                beforeAgent true
                expression { params.DISTRO_VERSION.toLowerCase().contains('ubuntu') }
                expression { params.ENABLED_STAGES.contains('build_artifacts') }
              }
              agent {
                node {
                  label 'ubuntu_kernel_builder'
                }
              }
              steps {
              withCredentials(bindings: [string(credentialsId: 'KERNEL_GIT_URL',
                                  variable: 'KERNEL_GIT_URL')]) {
                  stash includes: 'scripts/package_building/kernel_versions.ini', name: 'kernel_version_ini'
                  sh '''#!/bin/bash
                    set -xe
                    echo "Building artifacts..."
                    pushd "$WORKSPACE/scripts/package_building"
                    bash build_artifacts.sh \\
                        --git_url "${KERNEL_GIT_URL}" \\
                        --git_branch "${KERNEL_GIT_BRANCH}" \\
                        --destination_path "${BUILD_NUMBER}-${BRANCH_NAME}-${KERNEL_ARTIFACTS_PATH}" \\
                        --install_deps "True" \\
                        --thread_number "x3" \\
                        --debian_os_version "16" \\
                        --build_path "${BUILD_PATH}" \\
                        --kernel_config "${KERNEL_CONFIG}" \\
                        --clean_env "${CLEAN_ENV}" \\
                        --use_ccache "${USE_CCACHE}" \\
                        --enable_kernel_debug "${KERNEL_DEBUG}"
                    popd
                '''
              }

              sh '''#!/bin/bash
                  echo ${BUILD_NUMBER}-$(crudini --get scripts/package_building/kernel_versions.ini KERNEL_BUILT folder) > ./build_name
                '''
                script {
                  currentBuild.displayName = readFile "./build_name"
                }
                stash includes: 'scripts/package_building/kernel_versions.ini', name: 'kernel_version_ini'
                stash includes: ("scripts/package_building/${env.BUILD_NUMBER}-${env.BRANCH_NAME}-${env.KERNEL_ARTIFACTS_PATH}/**/deb/**"),
                      name: "${env.KERNEL_ARTIFACTS_PATH}"
                sh '''
                    set -xe
                    rm -rf "scripts/package_building/${BUILD_NUMBER}-${BRANCH_NAME}-${KERNEL_ARTIFACTS_PATH}"
                '''
                archiveArtifacts 'scripts/package_building/kernel_versions.ini'
              }
              post {
                failure {
                  reportStageStatus("BuildSucceeded", 0)
                }
                success {
                  reportStageStatus("BuildSucceeded", 1)
                }
              }
          }
          stage('build_artifacts_centos') {
              when {
                beforeAgent true
                expression { params.DISTRO_VERSION.toLowerCase().contains('centos') }
                expression { params.ENABLED_STAGES.contains('build_artifacts') }
              }
              agent {
                node {
                  label 'centos_kernel_builder'
                }
              }
              steps {
                withCredentials(bindings: [string(credentialsId: 'KERNEL_GIT_URL',
                                  variable: 'KERNEL_GIT_URL')]) {
                  stash includes: 'scripts/package_building/kernel_versions.ini', name: 'kernel_version_ini'
                  sh '''#!/bin/bash
                    set -xe
                    echo "Building artifacts..."
                    pushd "$WORKSPACE/scripts/package_building"
                    bash build_artifacts.sh \\
                        --git_url "${KERNEL_GIT_URL}" \\
                        --git_branch "${KERNEL_GIT_BRANCH}" \\
                        --destination_path "${BUILD_NUMBER}-${BRANCH_NAME}-${KERNEL_ARTIFACTS_PATH}" \\
                        --install_deps "True" \\
                        --thread_number "x3" \\
                        --build_path "${BUILD_PATH}" \\
                        --kernel_config "${KERNEL_CONFIG}" \\
                        --clean_env "${CLEAN_ENV}" \\
                        --use_ccache "${USE_CCACHE}" \\
                        --enable_kernel_debug "${KERNEL_DEBUG}"
                    popd
                '''
                }
                sh '''#!/bin/bash
                  echo ${BUILD_NUMBER}-$(crudini --get scripts/package_building/kernel_versions.ini KERNEL_BUILT folder) > ./build_name
                '''
                script {
                  currentBuild.displayName = readFile "./build_name"
                }
                stash includes: 'scripts/package_building/kernel_versions.ini', name: 'kernel_version_ini'
                stash includes: ("scripts/package_building/${env.BUILD_NUMBER}-${env.BRANCH_NAME}-${env.KERNEL_ARTIFACTS_PATH}/**/rpm/**"),
                      name: "${env.KERNEL_ARTIFACTS_PATH}"
                sh '''
                    set -xe
                    rm -rf "scripts/package_building/${BUILD_NUMBER}-${BRANCH_NAME}-${KERNEL_ARTIFACTS_PATH}"
                '''
                archiveArtifacts 'scripts/package_building/kernel_versions.ini'
              }
              post {
                failure {
                  reportStageStatus("BuildSucceeded", 0)
                }
                success {
                  reportStageStatus("BuildSucceeded", 1)
                }
              }
    }
    stage('publish_temp_artifacts') {
      when {
        beforeAgent true
        expression { params.ENABLED_STAGES.contains('publish_temp_artifacts') }
      }
      agent {
        node {
          label 'meta_slave'
        }
      }
      steps {
        dir("${env.KERNEL_ARTIFACTS_PATH}${env.BUILD_NUMBER}${env.BRANCH_NAME}") {
            unstash "${env.KERNEL_ARTIFACTS_PATH}"
            withCredentials([string(credentialsId: 'SMB_SHARE_URL', variable: 'SMB_SHARE_URL'),
                               usernamePassword(credentialsId: 'smb_share_user_pass',
                                                passwordVariable: 'PASSWORD',
                                                usernameVariable: 'USERNAME')]) {
                sh '''#!/bin/bash
                    set -xe
                    bash "${WORKSPACE}/scripts/utils/publish_artifacts_to_smb.sh" \\
                        --build_number "${BUILD_NUMBER}-${BRANCH_NAME}" \\
                        --smb_url "${SMB_SHARE_URL}/temp-kernel-artifacts" --smb_username "${USERNAME}" \\
                        --smb_password "${PASSWORD}" --artifacts_path "${KERNEL_ARTIFACTS_PATH}" \\
                        --artifacts_folder_prefix "${FOLDER_PREFIX}"
                '''
            }
        }
      }
    }
    stage('boot_test') {
      when {
        beforeAgent true
        expression { params.ENABLED_STAGES.contains('boot_test') }
      }
      post {
        failure {
          reportStageStatus("BootOnAzure", 0)
        }
        success {
          reportStageStatus("BootOnAzure", 1)
        }
      }
      parallel {
        stage('boot_test_large') {
            when {
              beforeAgent true
              expression { params.ENABLED_STAGES.contains('boot_test_large') }
            }
            agent {
              node {
                label 'azure'
              }
            }
            steps {
                withCredentials(bindings: [
                  file(credentialsId: 'Azure_Secrets_TESTONLY_File',
                       variable: 'Azure_Secrets_File')
                ]) {
                    prepareEnv(LISAV2_BRANCH, LISAV2_REMOTE, DISTRO_VERSION)
                    unstashKernel(env.KERNEL_ARTIFACTS_PATH)
                    RunPowershellCommand(".\\Run-LisaV2.ps1" +
                        " -TestLocation '${LISAV2_AZURE_REGION}'" +
                        " -RGIdentifier '${env.LISAV2_RG_IDENTIFIER}'" +
                        " -TestPlatform 'Azure'" +
                        " -CustomKernel 'localfile:./scripts/package_building/${env.BUILD_NUMBER}-${env.BRANCH_NAME}-${env.KERNEL_ARTIFACTS_PATH}/*/${env.PACKAGE_TYPE}/*.${env.PACKAGE_TYPE}'" +
                        " -OverrideVMSize '${env.LISAV2_AZURE_VM_SIZE_LARGE}'" +
                        " -ARMImageName '${env.AZURE_OS_IMAGE}'" +
                        " -TestNames 'VERIFY-LIS-MODULES-VERSION'" +
                        " -StorageAccount 'ExistingStorage_Standard'" +
                        " -XMLSecretFile '${env.Azure_Secrets_File}'" +
                        " -CustomParameters 'DiskType = Managed'"
                    )
                }
            }
            post {
              always {
                junit "Report\\*-junit.xml"
                archiveArtifacts "TestResults\\**\\*"
              }
            }
        }
        stage('boot_test_small') {
            agent {
              node {
                label 'azure'
              }
            }
            steps {
                withCredentials(bindings: [
                  file(credentialsId: 'Azure_Secrets_TESTONLY_File',
                       variable: 'Azure_Secrets_File')
                ]) {
                    prepareEnv(LISAV2_BRANCH, LISAV2_REMOTE, DISTRO_VERSION)
                    unstashKernel(env.KERNEL_ARTIFACTS_PATH)
                    RunPowershellCommand(".\\Run-LisaV2.ps1" +
                        " -TestLocation '${LISAV2_AZURE_REGION}'" +
                        " -RGIdentifier '${env.LISAV2_RG_IDENTIFIER}'" +
                        " -TestPlatform 'Azure'" +
                        " -CustomKernel 'localfile:./scripts/package_building/${env.BUILD_NUMBER}-${env.BRANCH_NAME}-${env.KERNEL_ARTIFACTS_PATH}/*/${env.PACKAGE_TYPE}/*.${env.PACKAGE_TYPE}'" +
                        " -OverrideVMSize '${env.LISAV2_AZURE_VM_SIZE_SMALL}'" +
                        " -ARMImageName '${env.AZURE_OS_IMAGE}'" +
                        " -TestNames 'VERIFY-LIS-MODULES-VERSION'" +
                        " -StorageAccount 'ExistingStorage_Standard'" +
                        " -XMLSecretFile '${env.Azure_Secrets_File}'" +
                        " -CustomParameters 'DiskType = Managed'"
                    )
                }
            }
            post {
              always {
                junit "Report\\*-junit.xml"
                archiveArtifacts "TestResults\\**\\*"
              }
            }
          }
      }
    }
    stage('publish_artifacts') {
      when {
        beforeAgent true
        expression { params.ENABLED_STAGES.contains('publish_artifacts') }
      }
      agent {
        node {
          label 'meta_slave'
        }
      }
      steps {
        dir("${env.KERNEL_ARTIFACTS_PATH}${env.BUILD_NUMBER}${env.BRANCH_NAME}") {
            unstash "${env.KERNEL_ARTIFACTS_PATH}"
            withCredentials([string(credentialsId: 'SMB_SHARE_URL', variable: 'SMB_SHARE_URL'),
                               usernamePassword(credentialsId: 'smb_share_user_pass', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')
                               ]) {
                sh '''#!/bin/bash
                    set -xe
                    bash "${WORKSPACE}/scripts/utils/publish_artifacts_to_smb.sh" \\
                        --build_number "${BUILD_NUMBER}-${BRANCH_NAME}" \\
                        --smb_url "${SMB_SHARE_URL}/${KERNEL_GIT_BRANCH_LABEL}-kernels" --smb_username "${USERNAME}" \\
                        --smb_password "${PASSWORD}" --artifacts_path "${KERNEL_ARTIFACTS_PATH}" \\
                        --artifacts_folder_prefix "${FOLDER_PREFIX}"
                '''
            }
        }
      }
    }
    stage('publish_azure_vhd') {
      when {
        beforeAgent true
        expression { params.ENABLED_STAGES.contains('publish_azure_vhd') }
      }
      agent {
        node {
          label 'azure'
        }
      }
      steps {
        withCredentials(bindings: [
          file(credentialsId: 'Azure_Secrets_TESTONLY_File',
               variable: 'Azure_Secrets_File')
        ]) {
            prepareEnv(LISAV2_BRANCH, LISAV2_REMOTE, DISTRO_VERSION)
            unstashKernel(env.KERNEL_ARTIFACTS_PATH)
            RunPowershellCommand(".\\Run-LisaV2.ps1" +
                " -TestLocation '${LISAV2_AZURE_REGION}'" +
                " -RGIdentifier '${env.LISAV2_RG_IDENTIFIER}'" +
                " -TestPlatform 'Azure'" +
                " -CustomKernel 'localfile:./scripts/package_building/${env.BUILD_NUMBER}-${env.BRANCH_NAME}-${env.KERNEL_ARTIFACTS_PATH}/*/${env.PACKAGE_TYPE}/*.${env.PACKAGE_TYPE}'" +
                " -OverrideVMSize '${env.LISAV2_AZURE_VM_SIZE_SMALL}'" +
                " -ARMImageName '${env.AZURE_OS_IMAGE}'" +
                " -TestNames 'CAPTURE-VHD-BEFORE-TEST'" +
                " -XMLSecretFile '${env.Azure_Secrets_File}'"
            )
            script {
                env.CapturedVHD = readFile 'CapturedVHD.azure.env'
            }
            stash includes: 'CapturedVHD.azure.env', name: 'CapturedVHD.azure.env'
            println("Captured VHD : ${env.CapturedVHD}")
        }
      }
      post {
        always {
          junit "Report\\*-junit.xml"
          archiveArtifacts "TestResults\\**\\*"
        }
      }
    }

    stage('validation') {
      parallel {

        stage('validation_hyperv') {
          when {
            beforeAgent true
            expression { params.ENABLED_STAGES.contains('validation_hyperv') }
          }
          agent {
            node {
              label 'hyper-v'
            }
          }
          steps {
            withCredentials(bindings: [
              file(credentialsId: 'HyperV_Secrets_File',
                   variable: 'HyperV_Secrets_File'),
              string(credentialsId: 'LISAV2_IMAGES_SHARE_URL',
                   variable: 'LISAV2_IMAGES_SHARE_URL')
            ]) {
            script {
                hashtableHV=[:]
                hashtableHV=getTests("${PRIORITY}", "HyperV")
                def stepsForParallel = [:]
                hashtableHV.each {
                    def stepName = it.key
                    def test_cmd = it.value
                    stepsForParallel[stepName] = { ->
                        node('hyper-v') {
                        stage("${stepName}") {
                            dir ("..\\${DISTRO_VERSION}-${env.BUILD_NUMBER}-${env.BRANCH_NAME}-${stepName}") {
                                deleteDir()
                                prepareEnv(LISAV2_BRANCH, LISAV2_REMOTE, DISTRO_VERSION)
                                unstashKernel(env.KERNEL_ARTIFACTS_PATH)
                                script {
                                env.HYPERV_VHD_PATH = getVhdLocation(LISAV2_IMAGES_SHARE_URL, DISTRO_VERSION)
                                }
                                println("Current VHD: ${env.HYPERV_VHD_PATH}")
                                try {
                                    RunPowershellCommand(".\\Run-LisaV2.ps1" +
                                    " -TestLocation 'localhost'" +
                                    " -RGIdentifier '${LISAV2_RG_IDENTIFIER}'" +
                                    " -TestPlatform 'HyperV'" +
                                    " ${test_cmd}" +
                                    " -CustomKernel 'localfile:./scripts/package_building/${env.BUILD_NUMBER}-${env.BRANCH_NAME}-${env.KERNEL_ARTIFACTS_PATH}/*/${env.PACKAGE_TYPE}/*.${env.PACKAGE_TYPE}'" +
                                    " -OsVHD '${HYPERV_VHD_PATH}'" +
                                    " -XMLSecretFile '${HyperV_Secrets_File}'" +
                                    " -ResourceCleanup Delete" +
                                    " -ExitWithZero")
                                } finally {
                                    junit "Report\\*-junit.xml"
                                    archiveArtifacts "*-TestLogs.zip"
                                    script {
                                        def passCount = GetTestResults("pass")
                                        def abortCount = GetTestResults("abort")
                                        def failCount = GetTestResults("fail")
                                        def skippedCount = GetTestResults("skipped")
                                        FUNC_PASS_ONLOCAL = FUNC_PASS_ONLOCAL.toInteger() + passCount.toInteger()
                                        FUNC_ABORT_ONLOCAL = FUNC_ABORT_ONLOCAL.toInteger() + abortCount.toInteger()
                                        FUNC_FAIL_ONLOCAL = FUNC_FAIL_ONLOCAL.toInteger() + failCount.toInteger()
                                        FUNC_SKIP_ONLOCAL = FUNC_SKIP_ONLOCAL.toInteger() + skippedCount.toInteger()
                                    }
                                }
                                deleteDir()
                            }
                        }
                        }
                    }
                }
                parallel stepsForParallel
                }
            }
          }
        }

        stage('validation_sriov_hyperv') {
          when {
            beforeAgent true
            expression { params.ENABLED_STAGES.contains('validation_sriov_hyperv') }
          }
          agent {
            node {
              label 'sriov_pipelines'
            }
          }
          steps {
            withCredentials(bindings: [
              file(credentialsId: 'HyperV_Secrets_File',
                   variable: 'HyperV_Secrets_File'),
              string(credentialsId: 'LISAV2_IMAGES_SHARE_URL',
                   variable: 'LISAV2_IMAGES_SHARE_URL'),
              string(credentialsId:'SRIOV_TEST_LOCATION',
                   variable: 'SRIOV_TEST_LOCATION')
            ]) {
            script {
                hashtableSRIOV=[:]
                hashtableSRIOV=getTests("${PRIORITY}-SRIOV", "HyperV")
                def stepsForParallel = [:]
                hashtableSRIOV.each {
                    def stepName = it.key
                    def test_cmd = it.value
                    stepsForParallel[stepName] = { ->
                        stage("${stepName}") {
                            dir ("..\\${DISTRO_VERSION}-${env.BUILD_NUMBER}-${env.BRANCH_NAME}-${stepName}") {
                                deleteDir()
                                prepareEnv(LISAV2_BRANCH, LISAV2_REMOTE, DISTRO_VERSION)
                                unstashKernel(env.KERNEL_ARTIFACTS_PATH)
                                script {
                                    env.HYPERV_VHD_PATH = getVhdLocation(LISAV2_IMAGES_SHARE_URL, DISTRO_VERSION)
                                }
                                println("Current VHD: ${env.HYPERV_VHD_PATH}")
                                try {
                                    RunPowershellCommand(".\\Run-LisaV2.ps1" +
                                    " -TestLocation '${SRIOV_TEST_LOCATION}'" +
                                    " -RGIdentifier '${LISAV2_RG_IDENTIFIER}'" +
                                    " -TestPlatform 'HyperV'" +
                                    " ${test_cmd}" +
                                    " -CustomKernel 'localfile:./scripts/package_building/${env.BUILD_NUMBER}-${env.BRANCH_NAME}-${env.KERNEL_ARTIFACTS_PATH}/*/${env.PACKAGE_TYPE}/*.${env.PACKAGE_TYPE}'" +
                                    " -OsVHD '${HYPERV_VHD_PATH}'" +
                                    " -XMLSecretFile '${HyperV_Secrets_File}'" +
                                    " -ResourceCleanup Delete" +
                                    " -ExitWithZero")
                                } finally {
                                    junit "Report\\*-junit.xml"
                                    archiveArtifacts "*-TestLogs.zip"
                                    script {
                                        def passCount = GetTestResults("pass")
                                        def abortCount = GetTestResults("abort")
                                        def failCount = GetTestResults("fail")
                                        def skippedCount = GetTestResults("skipped")
                                        FUNC_PASS_ONLOCAL = FUNC_PASS_ONLOCAL.toInteger() + passCount.toInteger()
                                        FUNC_ABORT_ONLOCAL = FUNC_ABORT_ONLOCAL.toInteger() + abortCount.toInteger()
                                        FUNC_FAIL_ONLOCAL = FUNC_FAIL_ONLOCAL.toInteger() + failCount.toInteger()
                                        FUNC_SKIP_ONLOCAL = FUNC_SKIP_ONLOCAL.toInteger() + skippedCount.toInteger()
                                    }
                                }
                                deleteDir()
                            }
                        }
                    }
                }
                parallel stepsForParallel
                }
            }
          }
        }

        stage('validation_nvme_hyperv') {
          when {
            beforeAgent true
            expression { params.ENABLED_STAGES.contains('validation_nvme_hyperv') }
          }
          agent {
            node {
              label 'hy_nvme'
            }
          }
          steps {
            withCredentials(bindings: [
              file(credentialsId: 'HyperV_Secrets_File',
                   variable: 'HyperV_Secrets_File'),
              string(credentialsId: 'LISAV2_IMAGES_SHARE_URL',
                   variable: 'LISAV2_IMAGES_SHARE_URL')
            ]) {
            script {
                hashtableNVME=[:]
                hashtableNVME=getTests("${PRIORITY}-NVME", "HyperV")
                def stepsForParallel = [:]
                hashtableNVME.each {
                    def stepName = it.key
                    def test_cmd = it.value
                    stepsForParallel[stepName] = { ->
                        stage("${stepName}") {
                            dir ("..\\${DISTRO_VERSION}-${env.BUILD_NUMBER}-${env.BRANCH_NAME}-${stepName}") {
                                deleteDir()
                                prepareEnv(LISAV2_BRANCH, LISAV2_REMOTE, DISTRO_VERSION)
                                unstashKernel(env.KERNEL_ARTIFACTS_PATH)
                                script {
                                    env.HYPERV_VHD_PATH = getVhdLocation(LISAV2_IMAGES_SHARE_URL, DISTRO_VERSION)
                                }
                                println("Current VHD: ${env.HYPERV_VHD_PATH}")
                                try {
                                    RunPowershellCommand(".\\Run-LisaV2.ps1" +
                                    " -TestLocation 'localhost'" +
                                    " -RGIdentifier '${LISAV2_RG_IDENTIFIER}'" +
                                    " -TestPlatform 'HyperV'" +
                                    " ${test_cmd}" +
                                    " -CustomKernel 'localfile:./scripts/package_building/${env.BUILD_NUMBER}-${env.BRANCH_NAME}-${env.KERNEL_ARTIFACTS_PATH}/*/${env.PACKAGE_TYPE}/*.${env.PACKAGE_TYPE}'" +
                                    " -OsVHD '${HYPERV_VHD_PATH}'" +
                                    " -XMLSecretFile '${HyperV_Secrets_File}'" +
                                    " -ResourceCleanup Delete" +
                                    " -ExitWithZero")
                                } finally {
                                    junit "Report\\*-junit.xml"
                                    archiveArtifacts "*-TestLogs.zip"
                                    script {
                                            def passCount = GetTestResults("pass")
                                            def abortCount = GetTestResults("abort")
                                            def failCount = GetTestResults("fail")
                                            def skippedCount = GetTestResults("skipped")
                                            FUNC_PASS_ONLOCAL = FUNC_PASS_ONLOCAL.toInteger() + passCount.toInteger()
                                            FUNC_ABORT_ONLOCAL = FUNC_ABORT_ONLOCAL.toInteger() + abortCount.toInteger()
                                            FUNC_FAIL_ONLOCAL = FUNC_FAIL_ONLOCAL.toInteger() + failCount.toInteger()
                                            FUNC_SKIP_ONLOCAL = FUNC_SKIP_ONLOCAL.toInteger() + skippedCount.toInteger()
                                    }
                                }
                                deleteDir()
                            }
                        }
                    }
                }
                parallel stepsForParallel
                }
            }
          }
        }

        stage('validation_migrate_hyperv') {
          when {
            beforeAgent true
            expression { params.ENABLED_STAGES.contains('validation_migrate_hyperv') }
          }
          agent {
            node {
              label 'live_migration'
            }
          }
          steps {
            withCredentials(bindings: [
              file(credentialsId: 'HyperV_Secrets_File',
                   variable: 'HyperV_Secrets_File'),
              string(credentialsId: 'LISAV2_IMAGES_SHARE_URL',
                   variable: 'LISAV2_IMAGES_SHARE_URL')
            ]) {
            script {
                hashtableMIGRATE=[:]
                hashtableMIGRATE=getTests("${PRIORITY}-MIGRATE", "HyperV")
                def stepsForParallel = [:]
                hashtableMIGRATE.each {
                    def stepName = it.key
                    def test_cmd = it.value
                    stepsForParallel[stepName] = { ->
                        stage("${stepName}") {
                            dir ("..\\${DISTRO_VERSION}-${env.BUILD_NUMBER}-${env.BRANCH_NAME}-${stepName}") {
                                deleteDir()
                                prepareEnv(LISAV2_BRANCH, LISAV2_REMOTE, DISTRO_VERSION)
                                unstashKernel(env.KERNEL_ARTIFACTS_PATH)
                                script {
                                    env.HYPERV_VHD_PATH = getVhdLocation(LISAV2_IMAGES_SHARE_URL, DISTRO_VERSION)
                                }
                                println("Current VHD: ${env.HYPERV_VHD_PATH}")
                                try {
                                    RunPowershellCommand(".\\Run-LisaV2.ps1" +
                                    " -TestLocation 'localhost'" +
                                    " -RGIdentifier '${LISAV2_RG_IDENTIFIER}'" +
                                    " -TestPlatform 'HyperV'" +
                                    " ${test_cmd}" +
                                    " -CustomKernel 'localfile:./scripts/package_building/${env.BUILD_NUMBER}-${env.BRANCH_NAME}-${env.KERNEL_ARTIFACTS_PATH}/*/${env.PACKAGE_TYPE}/*.${env.PACKAGE_TYPE}'" +
                                    " -OsVHD '${HYPERV_VHD_PATH}'" +
                                    " -XMLSecretFile '${HyperV_Secrets_File}'" +
                                    " -ResourceCleanup Delete" +
                                    " -ExitWithZero")
                                } finally {
                                    junit "Report\\*-junit.xml"
                                    archiveArtifacts "*-TestLogs.zip"
                                    script {
                                        def passCount = GetTestResults("pass")
                                        def abortCount = GetTestResults("abort")
                                        def failCount = GetTestResults("fail")
                                        def skippedCount = GetTestResults("skipped")
                                        FUNC_PASS_ONLOCAL = FUNC_PASS_ONLOCAL.toInteger() + passCount.toInteger()
                                        FUNC_ABORT_ONLOCAL = FUNC_ABORT_ONLOCAL.toInteger() + abortCount.toInteger()
                                        FUNC_FAIL_ONLOCAL = FUNC_FAIL_ONLOCAL.toInteger() + failCount.toInteger()
                                        FUNC_SKIP_ONLOCAL = FUNC_SKIP_ONLOCAL.toInteger() + skippedCount.toInteger()
                                    }
                                }
                                deleteDir()
                            }
                        }
                    }
                }
                parallel stepsForParallel
            }
            }
          }
        }

        stage('validation_azure') {
          when {
            beforeAgent true
            expression { params.ENABLED_STAGES.contains('validation_azure') }
          }
          agent {
            node {
              label 'azure'
            }
          }
          steps {
            withCredentials(bindings: [
              file(credentialsId: 'Azure_Secrets_TESTONLY_File',
                   variable: 'Azure_Secrets_File')
            ]) {
            script {
                hashtableAZ=[:]
                hashtableAZ=getTests("${PRIORITY}", "Azure")
                def stepsForParallel = [:]
                hashtableAZ.each {
                    def stepName = it.key
                    def test_cmd = it.value
                    stepsForParallel[stepName] = { ->
                        node('azure') {
                        stage("${stepName}") {
                            dir ("..\\${DISTRO_VERSION}-${env.BUILD_NUMBER}-${env.BRANCH_NAME}-${stepName}") {
                                deleteDir()
                                prepareEnv(LISAV2_BRANCH, LISAV2_REMOTE, DISTRO_VERSION)
                                unstash 'CapturedVHD.azure.env'
                                script {
                                    env.CapturedVHD = readFile 'CapturedVHD.azure.env'
                                }
                                println("VHD under test : ${env.CapturedVHD}")
                                try {
                                    RunPowershellCommand(".\\Run-LisaV2.ps1" +
                                    " -TestLocation '${LISAV2_AZURE_REGION}'" +
                                    " -RGIdentifier '${LISAV2_RG_IDENTIFIER}'" +
                                    " -TestPlatform 'Azure'" +
                                    " ${test_cmd}" +
                                    " -OsVHD '${CapturedVHD}'" +
                                    " -XMLSecretFile '${Azure_Secrets_File}'" +
                                    " -ResourceCleanup Delete" +
                                    " -ExitWithZero")
                                } finally {
                                    junit "Report\\*-junit.xml"
                                    archiveArtifacts "*-TestLogs.zip"
                                    script {
                                        def passCount = GetTestResults("pass")
                                        def abortCount = GetTestResults("abort")
                                        def failCount = GetTestResults("fail")
                                        def skippedCount = GetTestResults("skipped")
                                        FUNC_PASS_ONAZURE = FUNC_PASS_ONAZURE.toInteger() + passCount.toInteger()
                                        FUNC_ABORT_ONAZURE = FUNC_ABORT_ONAZURE.toInteger() + abortCount.toInteger()
                                        FUNC_FAIL_ONAZURE = FUNC_FAIL_ONAZURE.toInteger() + failCount.toInteger()
                                        FUNC_SKIP_ONAZURE = FUNC_SKIP_ONAZURE.toInteger() + skippedCount.toInteger()
                                    }
                                }
                                deleteDir()
                            }
                        }
                        }
                    }
                }
                parallel stepsForParallel
                }
            }
          }
        }


      }
    }

    stage('publish_results') {
      when {
        beforeAgent true
        expression { params.ENABLED_STAGES.contains('publish_results') }
      }
      agent {
        node {
          label 'meta_slave'
        }
      }
      steps {
        reportStageStatus("FuncTestsFailedOnLocal", "${FUNC_FAIL_ONLOCAL}")
        reportStageStatus("FuncTestsFailedOnAzure", "${FUNC_FAIL_ONAZURE}")
        reportStageStatus("FuncTestsPassOnLocal", "${FUNC_PASS_ONLOCAL}")
        reportStageStatus("FuncTestsPassOnAzure", "${FUNC_PASS_ONAZURE}")
        reportStageStatus("FuncTestsAbortOnLocal", "${FUNC_ABORT_ONLOCAL}")
        reportStageStatus("FuncTestsAbortOnAzure", "${FUNC_ABORT_ONAZURE}")
        reportStageStatus("FuncTestsSkippedOnAzure", "${FUNC_SKIP_ONAZURE}")
        reportStageStatus("FuncTestsSkippedOnLocal", "${FUNC_SKIP_ONLOCAL}")
      }
    }

  }
}
