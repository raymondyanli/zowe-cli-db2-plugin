/*
* This program and the accompanying materials are made available under the terms of the *
* Eclipse Public License v2.0 which accompanies this distribution, and is available at *
* https://www.eclipse.org/legal/epl-v20.html                                      *
*                                                                                 *
* SPDX-License-Identifier: EPL-2.0                                                *
*                                                                                 *
* Copyright Contributors to the Zowe Project.                                     *
*                                                                                 *
*/

@Library('shared-pipelines') import org.zowe.pipelines.nodejs.NodeJSPipeline

import org.zowe.pipelines.nodejs.models.SemverLevel

/**
 * This is the product name used by the build machine to store information about
 * the builds
 */
def PRODUCT_NAME = "Zowe CLI"

node('ca-jenkins-agent') {
    // Initialize the pipeline
    def pipeline = new NodeJSPipeline(this)

    // Build admins, users that can approve the build and receieve emails for 
    // all protected branch builds.
    pipeline.admins.add("zfernand0", "mikebauerca", "markackert", "dkelosky")

    // Protected branch property definitions
    pipeline.protectedBranches.addMap([
        [name: "master", tag: "daily", prerelease: "alpha", dependencies: ["@zowe/imperative": "daily"]],
        [name: "beta", tag: "beta", prerelease: "beta", dependencies: ["@zowe/imperative": "beta"]],
        [name: "latest", tag: "latest", dependencies: ["@zowe/imperative": "latest"], autoDeploy: true],
        [name: "lts-incremental", tag: "lts-incremental", level: SemverLevel.MINOR, dependencies: ["@brightside/imperative": "lts-incremental"]],
        [name: "lts-stable", tag: "lts-stable", level: SemverLevel.PATCH, dependencies: ["@brightside/imperative": "lts-stable"]]
    ])

    // Git configuration information
    pipeline.gitConfig = [
        email: 'zowe.robot@gmail.com',
        credentialsId: 'zowe-robot-github'
    ]

    // npm publish configuration
    pipeline.publishConfig = [
        email: pipeline.gitConfig.email,
        credentialsId: 'GizaArtifactory',
        scope: '@zowe'
    ]

    pipeline.registryConfig = [
        [
            email: pipeline.publishConfig.email,
            credentialsId: pipeline.publishConfig.credentialsId,
            url: 'https://gizaartifactory.jfrog.io/gizaartifactory/api/npm/npm-release/',
            scope: pipeline.publishConfig.scope
        ]
    ]

    // Initialize the pipeline library, should create 5 steps
    pipeline.setup()

    // Create a custom lint stage that runs immediately after the setup.
    pipeline.createStage(
        name: "Lint",
        stage: {
            sh "npm run lint"
        },
        timeout: [
            time: 2,
            unit: 'MINUTES'
        ]
    )

    // Build the application
    pipeline.build(timeout: [
        time: 5,
        unit: 'MINUTES'
    ])

    def TEST_ROOT = "__tests__/__results__"
    def UNIT_TEST_ROOT = "$TEST_ROOT/unit"
    def UNIT_JUNIT_OUTPUT = "$UNIT_TEST_ROOT/junit.xml"
    
    // Perform a unit test and capture the results
    pipeline.test(
        name: "Unit",
        operation: {
            sh "npm run test:unit"
        },
        environment: [
            JEST_JUNIT_OUTPUT: UNIT_JUNIT_OUTPUT,
            JEST_SUIT_NAME: "Unit Tests",
            JEST_JUNIT_ANCESTOR_SEPARATOR: " > ",
            JEST_JUNIT_CLASSNAME: "Unit.{classname}",
            JEST_JUNIT_TITLE: "{title}",
            JEST_STARE_RESULT_DIR: "${UNIT_TEST_ROOT}/jest-stare",
            JEST_STARE_RESULT_HTML: "index.html"
        ],
        testResults: [dir: "${UNIT_TEST_ROOT}/jest-stare", files: "index.html", name: "${PRODUCT_NAME} - Unit Test Report"],
        coverageResults: [dir: "__tests__/__results__/unit/coverage/lcov-report", files: "index.html", name: "${PRODUCT_NAME} - Unit Test Coverage Report"],
        junitOutput: UNIT_JUNIT_OUTPUT,
        cobertura: [
            autoUpdateHealth: false,
            autoUpdateStability: false,
            coberturaReportFile: '__tests__/__results__/unit/coverage/cobertura-coverage.xml',
            classCoverageTargets: '85, 80, 75',
            conditionalCoverageTargets: '70, 65, 60',
            failUnhealthy: false,
            failUnstable: false,
            fileCoverageTargets: '100, 95, 80',
            lineCoverageTargets: '80, 70, 50',
            maxNumberOfBuilds: 20,
            methodCoverageTargets: '80, 70, 50',
            onlyStable: false,
            sourceEncoding: 'ASCII',
            zoomCoverageChart: false
        ]
    )

    // Check for vulnerabilities
    pipeline.checkVulnerabilities()

    // Deploys the application if on a protected branch. Give the version input
    // 30 minutes before an auto timeout approve.
    pipeline.deploy(
        versionArguments: [timeout: [time: 30, unit: 'MINUTES']]
    )

    // Once called, no stages can be added and all added stages will be executed. On completion
    // appropriate emails will be sent out by the shared library.
    pipeline.end()
}
