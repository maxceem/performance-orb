# Topcoder - JMeter Performance Testing Framework - Part 2

## Requirements
- [circleci](https://app.circleci.com/pipelines/github/topcoder-platform)
## Project structure
- jmeter
  - JsonSample.groovy
    - _Custom Json sampler in groovy_
- pom.xml
   - _Maven configuration file_
- .circleci
   - orb.yml
## What is provided in this project?


### Sensitive variables
Sensitive informations are now loaded though environments variables.
- User credentials
- M2M credentials
- TopCoder API URL
  - This one is not sensitive but loading it through env var instead of csv or json is more convenient.

Adding new one is as simple as adding new user variable within JMeter. The value of the variable has just to be set to : 
```groovy
${__groovy(System.getenv("ENV_VAR_NAME"))}
```
You can see exemples in this example JMeter [test file](https://github.com/codejamtc/resources-api/blob/performance/test/jmeter/Resource%20API.jmx)
- Authentication/Crendentials
- Resource API[DEV]/API Variables

As these env vars are copied to standard JMeter vars, they can be used anywhere in the test plan with `${var}`.

## Json datasets
A groovy script `JsonSample.groovy` is provided to load data from Json files.
It has been implemented using [these datasets](https://github.com/topcoder-platform/resources-api/tree/develop/test/postman/testData/resource-role) under `Create Resource Role`of the test plan.

The script is fully generic and can be used for any `json` file that has an array as the root object (see bellow).

### Format requirements
The JSon root object must be an array, the content of it is not restricted.
```json
[
  {
    "memberHandle": "handle1",
    "httpCode": 200
  },
  {
    "memberHandle": "handle2",
    "httpCode": 200
  }
]
```

### Usage
The has to be used as a `JSR223 Sampler` or `JSR223 PreProcessor` with groovy interpreter.

Here are the required parameters
- File Name: `JsonSample.groovy` (script path relative to test directory)

- Parameters: json dataset location + destination var, separated by space.
  - ie: /data/sample.json myvar

### Examples
You can see an exemple of using the parser as a Sample in the `create-resource-role-by-admin` section of the test plan

An exemple using it as a pre-processor is available with `create-resource-role-by-m2m`, an inline groovy script is then used to remove unneeded fileds from data, but it can also be written as a plain groovy script. See the body of the `Create active, read and write access resource role by M2M`request.
```groovy
${__groovy(
	import groovy.json.JsonOutput
	JsonOutput.toJson(vars.getObject("resource_role_m2m").findAll {k\,v -> !['httpCode'\, 'message'].contains(k)} )
)}
```

### Behavior

Each of the JMeter gets a different value, very similar to the CSV Dataset Behavior.

If there is no more data to load, the reading restarts at the beginning.

Each element of the Json array is stored as Object to the JMeter variable whose is name from the second groovy script parameter (see Parameters above).

The result is also stored as the `SampleResult` if used as a Sample and can be use by the orginal Jmeter parsers (ie. JsonPath extractor)

Finally the element json string is stored inside the `varname_json` var.

The result can alos be processed by any of the JMeter scripting languages, an exemple is provided in the JMeter test plan.

```groovy
// Extracting simple json object (no children) to vars named with the keys of the json data
def data = vars.getObject("tokens")
data.entrySet().each {
	vars.put(it.getKey(), it.getValue())
}
```



## Testing

## :bulb: Integration with CircleCI
#### Publishing this ORB :

Use [CircleCI CLI](https://circleci.com/docs/2.0/local-cli/) :

[Optional]  :information_source: Register a namespace if not already done : https://circleci.com/docs/2.0/orb-author-intro/#register-a-namespace

1. Create (**If not already created**) the orb under your namespace and orb name . I have used `topcoder-platform/performance-orb` where `topcoder-platform` is namespace and `performance-orb` is the orb name.

   ```
   circleci orb create topcoder-platform/performance-orb
   ```

   

2. Then publish it. (every time you update, publish with newer version)

    ```
    circleci orb publish .circleci/orb.yml topcoder-platform/performance- orb@1.0.0
    ```

:white_check_mark: If any other namespace/orb name/ version is used change that accordingly while importing the orb.

This can be [referred](https://circleci.com/developer/orbs/orb/sayantandey/test-orb4) as a sample. 

#### Importing this ORB :

To import and use this orb , import it in your circleci configuration like below under the root node. : 

format should be :

```yaml
orbs:
  orb-name: namespace/orb-name@version
```

For example :

```yaml
orbs:
  performance-orb: topcoder-platform/performance-orb@1.0.0
```



:bulb: According to this [documentation](https://circleci.com/docs/2.0/pipeline-variables/#the-scope-of-pipeline-parameters) : 

>  Pipeline parameters can only be resolved in the `.circleci/config.yml` file

So, the pipeline parameter `run_performancetesting` needs to be declared in individual config file and inside `parameter` field, like below :

```yaml
parameters:
  run_performancetesting:
    default: false
    type: boolean  
```

Sample HTML  [report](Challenge API Performance Test Report in HTML.PNG) : 

[Reference](https://circleci.com/docs/2.0/orb-intro/#benefits-of-using-orbs)

#### Additional Environment Variables :

Along with API's own circleci environment variables, the following environment varibales should be set in circleci. 

| Variable        | Explanation                                                  | Example                       |
| --------------- | ------------------------------------------------------------ | ----------------------------- |
| `SERVER_URL`    | Required. The base url of the Application which needs to be tested. | https://api.topcoder-dev.com  |
| `JOB_CLEAN_API` | Required. The cleanup API for that application. For example  | resources/internal/jobs/clean |
| `JMX_LOCATION`  | Required. Location of the .jmx file                          | ./test                        |
| `JMX_FILE`      | Required. Name of the jmx file to be used.                   | Resource API.jmx              |



Use a classic Maven/Java pipeline : [Language Guide: Java (with Maven)](https://circleci.com/docs/2.0/language-java-maven/)

Provided required environment variables : [Using Environment Variables](https://circleci.com/docs/2.0/env-vars/)

## Resources

[CircleCI Orb Docs](https://circleci.com/docs/2.0/orb-intro/#section=configuration) - Docs for using and creating CircleCI Orbs.

## Errors

All errors reported by JMeter where already present in the provided tests plan, it was not asked to solve them.
- User 2&3 Invalid credentials
- RessourceRole creation
  - Provided ids already exists, not modified to avoid creating millions of entries in the ressource-role database.
