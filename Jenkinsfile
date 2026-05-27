// Load shared library named 'jenkins-shared-library'
@Library('jenkins-shared-library') _

// Common configuration passed to the pipeline function
def configMap = [
    project  : "roboshop",   // Project name
    component: "catalogue"   // Service/component name
]

// Check current branch name
if (!env.BRANCH_NAME.equalsIgnoreCase('main')) {
    // For all non-main branches, run the EKS Node.js pipeline
    nodejsEKSPipeline(configMap)
} else {
    // For main branch, do not run this pipeline; show message to proceed with PROD
    echo "Please proceed with PROD process"
}
