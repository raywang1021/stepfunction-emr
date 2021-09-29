# Step Functions to Orchestrate Amazon EMR


Step Funcitons Amazon States Language (ASL):

```
{
  "StartAt": "Should_Create_Cluster",
  "States": {
    "Should_Create_Cluster": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.CreateCluster",
          "BooleanEquals": true,
          "Next": "Create_A_Cluster"
        },
        {
          "Variable": "$.CreateCluster",
          "BooleanEquals": false,
          "Next": "Enable_Termination_Protection"
        }
      ],
      "Default": "Create_A_Cluster"
    },
    "Create_A_Cluster": {
      "Type": "Task",
      "Resource": "arn:aws:states:::elasticmapreduce:createCluster.sync",
      "Parameters": {
        "Name": "WorkflowCluster",
        "VisibleToAllUsers": true,
        "ReleaseLabel": "emr-5.28.0",
        "Applications": [{ "Name": "Hive" }],
        "ServiceRole": "EMR_DefaultRole",
        "JobFlowRole": "EMR_EC2_DefaultRole",
        "LogUri": "s3://aws-logs-123412341234-eu-west-1/elasticmapreduce/",
        "Instances": {
          "KeepJobFlowAliveWhenNoSteps": true,
          "InstanceFleets": [
            {
              "InstanceFleetType": "MASTER",
              "TargetOnDemandCapacity": 1,
              "InstanceTypeConfigs": [
                {
                  "InstanceType": "m4.xlarge"
                }
              ]
            },
            {
              "InstanceFleetType": "CORE",
              "TargetOnDemandCapacity": 1,
              "InstanceTypeConfigs": [
                {
                  "InstanceType": "m4.xlarge"
                }
              ]
            }
          ]
        }
      },
      "ResultPath": "$.CreateClusterResult",
      "Next": "Merge_Results"
    },
    "Merge_Results": {
      "Type": "Pass",
      "Parameters": {
        "CreateCluster.$": "$.CreateCluster",
        "TerminateCluster.$": "$.TerminateCluster",
        "ClusterId.$": "$.CreateClusterResult.ClusterId"
      },
      "Next": "Enable_Termination_Protection"
    },
    "Enable_Termination_Protection": {
      "Type": "Task",
      "Resource": "arn:aws:states:::elasticmapreduce:setClusterTerminationProtection",
      "Parameters": {
        "ClusterId.$": "$.ClusterId",
        "TerminationProtected": true
      },
      "ResultPath": null,
      "Next": "Add_Steps_Parallel"
    },
    "Add_Steps_Parallel": {
      "Type": "Parallel",
      "Branches": [
        {
          "StartAt": "Step_One",
          "States": {
            "Step_One": {
              "Type": "Task",
              "Resource": "arn:aws:states:::elasticmapreduce:addStep.sync",
              "Parameters": {
                "ClusterId.$": "$.ClusterId",
                "Step": {
                  "Name": "The first step",
                  "ActionOnFailure": "CONTINUE",
                  "HadoopJarStep": {
                    "Jar": "command-runner.jar",
                    "Args": [
                      "hive-script",
                      "--run-hive-script",
                      "--args",
                      "-f",
                      "s3://eu-west-1.elasticmapreduce.samples/cloudfront/code/Hive_CloudFront.q",
                      "-d",
                      "INPUT=s3://eu-west-1.elasticmapreduce.samples",
                      "-d",
                      "OUTPUT=s3://MY-BUCKET/MyHiveQueryResults/"
                    ]
                  }
                }
              },
              "End": true
            }
          }
        },
        {
          "StartAt": "Wait_10_Seconds",
          "States": {
            "Wait_10_Seconds": {
              "Type": "Wait",
              "Seconds": 10,
              "Next": "Step_Two (async)"
            },
            "Step_Two (async)": {
              "Type": "Task",
              "Resource": "arn:aws:states:::elasticmapreduce:addStep",
              "Parameters": {
                "ClusterId.$": "$.ClusterId",
                "Step": {
                  "Name": "The second step",
                  "ActionOnFailure": "CONTINUE",
                  "HadoopJarStep": {
                    "Jar": "command-runner.jar",
                    "Args": [
                      "hive-script",
                      "--run-hive-script",
                      "--args",
                      "-f",
                      "s3://eu-west-1.elasticmapreduce.samples/cloudfront/code/Hive_CloudFront.q",
                      "-d",
                      "INPUT=s3://eu-west-1.elasticmapreduce.samples",
                      "-d",
                      "OUTPUT=s3://MY-BUCKET/MyHiveQueryResults/"
                    ]
                  }
                }
              },
              "ResultPath": "$.AddStepsResult",
              "Next": "Wait_Another_10_Seconds"
            },
            "Wait_Another_10_Seconds": {
              "Type": "Wait",
              "Seconds": 10,
              "Next": "Cancel_Step_Two"
            },
            "Cancel_Step_Two": {
              "Type": "Task",
              "Resource": "arn:aws:states:::elasticmapreduce:cancelStep",
              "Parameters": {
                "ClusterId.$": "$.ClusterId",
                "StepId.$": "$.AddStepsResult.StepId"
              },
              "End": true
            }
          }
        }
      ],
      "ResultPath": null,
      "Next": "Step_Three"
    },
    "Step_Three": {
      "Type": "Task",
      "Resource": "arn:aws:states:::elasticmapreduce:addStep.sync",
      "Parameters": {
        "ClusterId.$": "$.ClusterId",
        "Step": {
          "Name": "The third step",
          "ActionOnFailure": "CONTINUE",
          "HadoopJarStep": {
            "Jar": "command-runner.jar",
            "Args": [
              "hive-script",
              "--run-hive-script",
              "--args",
              "-f",
              "s3://eu-west-1.elasticmapreduce.samples/cloudfront/code/Hive_CloudFront.q",
              "-d",
              "INPUT=s3://eu-west-1.elasticmapreduce.samples",
              "-d",
              "OUTPUT=s3://MY-BUCKET/MyHiveQueryResults/"
            ]
          }
        }
      },
      "ResultPath": null,
      "Next": "Disable_Termination_Protection"
    },
    "Disable_Termination_Protection": {
      "Type": "Task",
      "Resource": "arn:aws:states:::elasticmapreduce:setClusterTerminationProtection",
      "Parameters": {
        "ClusterId.$": "$.ClusterId",
        "TerminationProtected": false
      },
      "ResultPath": null,
      "Next": "Should_Terminate_Cluster"
    },
    "Should_Terminate_Cluster": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.TerminateCluster",
          "BooleanEquals": true,
          "Next": "Terminate_Cluster"
        },
        {
          "Variable": "$.TerminateCluster",
          "BooleanEquals": false,
          "Next": "Wrapping_Up"
        }
      ],
      "Default": "Wrapping_Up"
    },
    "Terminate_Cluster": {
      "Type": "Task",
      "Resource": "arn:aws:states:::elasticmapreduce:terminateCluster.sync",
      "Parameters": {
        "ClusterId.$": "$.ClusterId"
      },
      "Next": "Wrapping_Up"
    },
    "Wrapping_Up": {
      "Type": "Pass",
      "End": true
    }
  }
}
```

Create a new EMR and terminate it:
```
{
  "CreateCluster": true,
  "TerminateCluster": true
}
```

To use an existing cluster:
```
{
  "CreateCluster": false,
  "TerminateCluster": false,
  "ClusterId": "j-xxx-yyy"
}
```
