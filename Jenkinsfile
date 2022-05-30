pipeline {
  agent any
  environment {
    PIPELINE_USER_CREDENTIAL_ID = 'aws-access'
    SAM_TEMPLATE = 'sam-app/template.yaml'
    MAIN_BRANCH = 'main'
    TESTING_STACK_NAME = 'sam-app-dev'
    TESTING_PIPELINE_EXECUTION_ROLE = 'arn:aws:iam::043576882669:role/aws-sam-cli-managed-dev-pipe-PipelineExecutionRole-NQCEYLF7FGDE'
    TESTING_CLOUDFORMATION_EXECUTION_ROLE = 'arn:aws:iam::043576882669:role/aws-sam-cli-managed-dev-p-CloudFormationExecutionR-W2Q4O6FKZKJP'
    TESTING_ARTIFACTS_BUCKET = 'aws-sam-cli-managed-dev-pipeline-artifactsbucket-7d9oiv4a7oqx'
    // If there are functions with "Image" PackageType in your template,
    // uncomment the line below and add "--image-repository ${TESTING_IMAGE_REPOSITORY}" to
    // testing "sam package" and "sam deploy" commands.
    // TESTING_IMAGE_REPOSITORY = '0123456789.dkr.ecr.region.amazonaws.com/repository-name'
    TESTING_REGION = 'us-west-2'
    PROD_STACK_NAME = 'sam-app-prod'
    PROD_PIPELINE_EXECUTION_ROLE = 'arn:aws:iam::043576882669:role/aws-sam-cli-managed-prod-pip-PipelineExecutionRole-YKXCCZM2CZR6'
    PROD_CLOUDFORMATION_EXECUTION_ROLE = 'arn:aws:iam::043576882669:role/aws-sam-cli-managed-prod-CloudFormationExecutionR-1U209BZ4E8MS'
    PROD_ARTIFACTS_BUCKET = 'aws-sam-cli-managed-prod-pipeline-artifactsbucket-1ivovi9osf4rf'
    // If there are functions with "Image" PackageType in your template,
    // uncomment the line below and add "--image-repository ${PROD_IMAGE_REPOSITORY}" to
    // prod "sam package" and "sam deploy" commands.
    // PROD_IMAGE_REPOSITORY = '0123456789.dkr.ecr.region.amazonaws.com/repository-name'
    PROD_REGION = 'us-west-2'
  }
  stages {
    // uncomment and modify the following step for running the unit-tests
    // stage('test') {
    //   steps {
    //     sh '''
    //       # trigger the tests here
    //     '''
    //   }
    // }

    stage('build-and-deploy-feature') {
      // this stage is triggered only for feature branches (feature*),
      // which will build the stack and deploy to a stack named with branch name.
      when {
        branch 'feature*'
      }
      agent {
        docker {
          image 'public.ecr.aws/sam/build-provided'
          args '--user 0:0 -v /var/run/docker.sock:/var/run/docker.sock'
        }
      }
      steps {
        sh 'sam build --template ${SAM_TEMPLATE} --use-container'
        withAWS(
            credentials: env.PIPELINE_USER_CREDENTIAL_ID,
            region: env.TESTING_REGION,
            role: env.TESTING_PIPELINE_EXECUTION_ROLE,
            roleSessionName: 'deploying-feature') {
          sh '''
            sam deploy --stack-name $(echo ${BRANCH_NAME} | tr -cd '[a-zA-Z0-9-]') \
              --capabilities CAPABILITY_IAM \
              --region ${TESTING_REGION} \
              --s3-bucket ${TESTING_ARTIFACTS_BUCKET} \
              --no-fail-on-empty-changeset \
              --role-arn ${TESTING_CLOUDFORMATION_EXECUTION_ROLE}
          '''
        }
      }
    }

    stage('build-and-package') {
      when {
        branch env.MAIN_BRANCH
      }
      agent {
        docker {
          image 'public.ecr.aws/sam/build-provided'
          args '--user 0:0 -v /var/run/docker.sock:/var/run/docker.sock'
        }
      }
      steps {
        sh 'sam build --template ${SAM_TEMPLATE} --use-container'
        withAWS(
            credentials: env.PIPELINE_USER_CREDENTIAL_ID,
            region: env.TESTING_REGION,
            role: env.TESTING_PIPELINE_EXECUTION_ROLE,
            roleSessionName: 'testing-packaging') {
          sh '''
            sam package \
              --s3-bucket ${TESTING_ARTIFACTS_BUCKET} \
              --region ${TESTING_REGION} \
              --output-template-file packaged-testing.yaml
          '''
        }

        withAWS(
            credentials: env.PIPELINE_USER_CREDENTIAL_ID,
            region: env.PROD_REGION,
            role: env.PROD_PIPELINE_EXECUTION_ROLE,
            roleSessionName: 'prod-packaging') {
          sh '''
            sam package \
              --s3-bucket ${PROD_ARTIFACTS_BUCKET} \
              --region ${PROD_REGION} \
              --output-template-file packaged-prod.yaml
          '''
        }

        archiveArtifacts artifacts: 'packaged-testing.yaml'
        archiveArtifacts artifacts: 'packaged-prod.yaml'
      }
    }

    stage('deploy-testing') {
      when {
        branch env.MAIN_BRANCH
      }
      agent {
        docker {
          image 'public.ecr.aws/sam/build-provided'
        }
      }
      steps {
        withAWS(
            credentials: env.PIPELINE_USER_CREDENTIAL_ID,
            region: env.TESTING_REGION,
            role: env.TESTING_PIPELINE_EXECUTION_ROLE,
            roleSessionName: 'testing-deployment') {
          sh '''
            sam deploy --stack-name ${TESTING_STACK_NAME} \
              --template packaged-testing.yaml \
              --capabilities CAPABILITY_IAM \
              --region ${TESTING_REGION} \
              --s3-bucket ${TESTING_ARTIFACTS_BUCKET} \
              --no-fail-on-empty-changeset \
              --role-arn ${TESTING_CLOUDFORMATION_EXECUTION_ROLE}
          '''
        }
      }
    }

    // uncomment and modify the following step for running the integration-tests
    // stage('integration-test') {
    //   when {
    //     branch env.MAIN_BRANCH
    //   }
    //   steps {
    //     sh '''
    //       # trigger the integration tests here
    //     '''
    //   }
    // }

    stage('deploy-prod') {
      when {
        branch env.MAIN_BRANCH
      }
      agent {
        docker {
          image 'public.ecr.aws/sam/build-provided'
        }
      }
      steps {
        // uncomment this to have a manual approval step before deployment to production
        // timeout(time: 24, unit: 'HOURS') {
        //   input 'Please confirm before starting production deployment'
        // }
        withAWS(
            credentials: env.PIPELINE_USER_CREDENTIAL_ID,
            region: env.PROD_REGION,
            role: env.PROD_PIPELINE_EXECUTION_ROLE,
            roleSessionName: 'prod-deployment') {
          sh '''
            sam deploy --stack-name ${PROD_STACK_NAME} \
              --template packaged-prod.yaml \
              --capabilities CAPABILITY_IAM \
              --region ${PROD_REGION} \
              --s3-bucket ${PROD_ARTIFACTS_BUCKET} \
              --no-fail-on-empty-changeset \
              --role-arn ${PROD_CLOUDFORMATION_EXECUTION_ROLE}
          '''
        }
      }
    }
  }
}
