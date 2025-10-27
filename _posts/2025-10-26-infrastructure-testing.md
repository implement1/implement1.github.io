# Why Untested Infrastructure is Broken Infrastructure: A GitHub Actions Journey

Countless production outages are caused by lack of testing: **infrastructure that isn't tested is broken until proven otherwise**.

Managing infrastructures using Terraform may seem to work after a successful `terraform apply` but more often than not, it is risky without proper testing. Automated testing often uncovers hidden issues that can cause production outages. Some of the issues I discovered during testing a sample lambda module are:

- **modules** with configuration drift issues
- **Security groups** misconfigurations
- **Lambda functions** failing to deploy properly in specific VPC configurations
- **Environment variables** corrupted during deployments

To ensure a reliable infrastructure code, a comprehensive testing strategy is required.

## Implementation

Here is an excerpt of a GitHub Actions workflow that automatically tests every component of the infrastructure code.
The GitHub Actions workflow implements 3 testing categories and each test runs in parallel to reduce feedback loop.

```yaml
name: VPC Lambda Tests
    runs-on: ubuntu-latest
    needs: validate
    if: needs.validate.outputs.should_run_tests == 'true' && (github.event.inputs.test_category == 'all' || github.event.inputs.test_category == 'vpc' || github.event.inputs.test_category == '')
    strategy:
      matrix:
        test_case:
          - TestLambdaVPCIntegration
          - TestLambdaVPCWithMultipleSubnets
          - TestLambdaVPCSecurityGroupRules      
```

### AWS Resource Testing

Unlike synthetic tests, this approach creates **AWS resources** during testing. Here's an example of the implementation:

```go
func TestLambdaBasicPython(t *testing.T) {
    // Generate unique names for resources
    uniqueID := random.UniqueId()
    functionName := fmt.Sprintf("test-lambda-basic-python-%s", uniqueID)
    
    // Terraform options
    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../examples/lambda-basic-python",
        Vars: map[string]interface{}{
            "function_name": functionName,
            "aws_region":    awsRegion,
        },
    })
    
    // Clean up resources with "terraform destroy" at the end
    defer terraform.Destroy(t, terraformOptions)
    
    // Run "terraform init" and "terraform apply"
    terraform.InitAndApply(t, terraformOptions)
    
    // Verify Lambda function exists and invoke it
    lambdaClient := aws.NewLambdaClient(t, awsRegion)
    invokeResult, err := lambdaClient.Invoke(&lambda.InvokeInput{
        FunctionName: awssdk.String(functionName),
        Payload:      []byte(`{"test": "data"}`),
    })
    require.NoError(t, err)
    assert.Equal(t, int64(200), *invokeResult.StatusCode)
}
```

### Test Coverage

The testing framework covers the following aspects for demo purposes:

- **Basic functionality**: Does the infrastructure actually work?
- **VPC integration**: Can Lambda functions communicate properly within VPCs?
- **Security configurations**: Are security groups configured correctly?
- **Performance benchmarks**: Does deployment complete within acceptable timeframes?
- **Environment variables**: Do configurations pass through properly?
- **Cleanup procedures**: Are resources properly destroyed?

### Resource Management

Every test should include automatic cleanup:

```yaml
cleanup:
  name: Cleanup Resources
  runs-on: ubuntu-latest
  needs: [test-basic, test-vpc, test-integration, test-performance]
  if: always() && needs.validate.outputs.should_run_tests == 'true'
  steps:
    - name: Cleanup test resources
      run: |
        # Clean up Lambda functions with test prefix
        aws lambda list-functions --query 'Functions[?starts_with(FunctionName, `test-lambda`)].FunctionName' --output text | \
        xargs -r -n1 aws lambda delete-function --function-name || true
```

## Results

- **Decreased Production incidents**
This enables everyone on the team to deploy infrastructure changes with confidence, because the tests catch issues before they become problems. 
- **Increase in Deployment confidence**
No more crossing fingers during deployments means less time spent debugging and more time spent building.
- **Cost Optimisation**
Catching security misconfigurations to prevent breaches is now automated.
Significant cost optimization by catching issues in testing.
![lambda-pipes](/Users/merciful/Pictures/lambdapipes1.png)


## Conclusion
Like any other code, Infrastructure code that is not tested is bound to have bugs. The question isn't whether infrastructure code should be tested—it's whether we can afford not to. There's no going back once you experience the peace of mind that comes with truly tested infrastructure.