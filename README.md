
# Testing with RAGAS

Repo contains test executor and test configs/results for evaluating LLM's output.

## Designing your tests

1. Create yaml test config file.
   
    Example:

    ```yaml
        test_suite_name: GenericSample
        test_executor_class_name: OpenWebUIRagasTestExecutor
        http_endpoint: https://dev.chat.soname.solutions
        login_credentials_ssm_param_name: param-sonamegpt-OpenWebUI-admin-creds-dev
        default_model_name: General AIAssistant
        response_streaming: true
        evaluators:
            - name: NovaPro-ScoreOutOfTen
              type: SimpleCriteriaScore
              evaluator_model_id: eu.amazon.nova-pro-v1:0
              definition: "Score 0 to 10 by correctness and completeness of the response. Check all the aspects of the reference answer."
        tests:
          - id: "S01"
            evaluator: NovaPro-ScoreOutOfTen
            question: What are main ingredients of sushi?
            reference_answer: |
              - Response must include mentions of Rice, Fish or seafood and Nori
            tags:
              - sushi-recipe
              - smoke

          - id: "S02"
            evaluator: NovaPro-ScoreOutOfTen
            question: Name the last 3 monarchs of the Great Britain
            reference_answer: |
              - Must include Charles 3rd, George 6th and Elizabeth 2nd    
    ```

    Config parameters:
    - `test_suite_name` - Decorative name. Test results filename will include this. Stick to alphanumeric characters.
    - `test_executor_class_name` - OpenWebUIRagasTestExecutor / BedrockAgentCoreRagasTestExecutor
    - `http_endpoint` - the URL of OpenWebUI instance / AgentCore Runtime API endpoint
    - `login_credentials_ssm_param_name` - Name of AWS SSM parameter containing login credentials for OpenWebUI API or Cognito
    - `default_model_name` - Default model used to generate responses in tests. Can be overriden in tests section
    - `response_streaming` - Where invocation should request streaming/no-streaming response
    - `evaluators` - define LLM models for scoring the actual response vs. reference answer
      - `name` - tests reference their evaluator by name
      - `type` - only SimpleCriteriaScore is supported
      - `evaluator_model_id` - LLM model name
      - `definition` - description how scoring should be done
      - `temperature` (optional) - default is 0.0 for more deterministic results
    - `tests` - can include multiple tests
    - each test structure:
      - `id` - unique test identifier
      - `evaluator` - reference to evaluator's name to use for response scoring
      - `model_name` (optional) - model to use to generate response (default = `default_model_name` from general section)
      - `question`
      - `reference_answer`
      - `tags` (optional) - list of string labels to categorize the test (e.g. `smoke`, `regression`). Used for filtering via `--tags`.

### Guidelines to design reference_answer

1. Describe all aspects which answer must contain.
2. For multiple aspects use bullet list. Numbered formatting leads to misinterpretation as ordered narrative.

## Running you tests

Prerequisites: 

1. `uv` installed ("pip install uv").
2. AWS CLI installed
3. AWS profile configured (it's used to retrieve OpenWebUI credentials from SSM parameter and to call Bedrock-based evaluator LLM).

`ragas_runner.py` uses subcommands. The two available commands are `run` and `list-tags`.

### `run` — execute a test suite

Sample command:

```bash
uv run ragas_runner.py run --config=configs/generic-sample.yaml
```

Parameters:

```text
  --config CONFIG       Path to config file (e.g., configs/generic-sample.yaml)
  --replacements        String to update placeholders in the config files. Must be provided in a form {'placeholder1' : 'val1'}
  --output_folder OUTPUT_FOLDER
                        Path to output folder (e.g., test_runs)
  --aws_profile_name AWS_PROFILE_NAME
                        AWS credentials profile name. For LLM evaluator and getting creds from AWS SSM
  --comments COMMENTS   Comments for the test run
  --questions QUESTIONS Comma-separated question IDs to run, supports ranges (e.g. B01,B02,B04-10). Runs all if not specified.
  --tags TAGS           Comma-separated tags to filter tests (e.g. smoke,regression). Runs tests matching ANY tag. Runs all if not specified.
  --filter-mode {and,or}
                        How to combine --questions and --tags filters: 'or' runs tests matching either (default), 'and' requires both.
```

- Default value for `aws_profile_name` (if omitted) - "SonameGPT-developer"
- `comments` - put descriptive comments, e.g. "running against Claude 4.5". It will be stored in test results file.
- `questions` - filter which tests to run by their `id`. Accepts comma-separated IDs and/or ranges. Examples:
  - `--questions=S01,S03` — runs only tests S01 and S03
  - `--questions=B01-05` — runs tests B01 through B05
  - `--questions=B01,B04-10` — runs B01 and B04 through B10
  - Omit the parameter to run all tests in the config.
- `tags` - filter which tests to run by their `tags`. Accepts comma-separated tag names. Examples:
  - `--tags=smoke` — runs only tests tagged `smoke`
  - `--tags=smoke,regression` — runs tests tagged with either `smoke` or `regression` (default `or` mode)
  - Omit the parameter to run all tests in the config.
- `filter-mode` - controls how `--questions` and `--tags` are combined when both are provided:
  - `--filter-mode=or` *(default)* — runs tests that match **either** the ID filter or the tag filter
  - `--filter-mode=and` — runs only tests that match **both** the ID filter and the tag filter

Test config files support placeholders. To execute tests dynamically:
  1. Test config files must be modified to enable placeholders in the format {{placeholder_name}}. E.g.,:
      ```yaml
        http_endpoint: https://{{brand}}-{{env}}.chat.soname.solutions/invocations?qualifier=DEFAULT
        login_credentials_ssm_param_name: param-sonamegpt-cognito-soname-user-{{env}}
      ```
  2. When executing the runner script, replacement argument must be provided in the following format:
      ```bash
      uv run ragas_runner.py run --config=configs/generic-sample.yaml --replacements="{'brand': 'bmw', 'env': 'dev'}"
      ```
## Running tests (MCON simplified)

In addition to the standard way of running tests using `--replacements`, a simplified approach was introduced specifically for MCON.

To run tests with the new configuration, you only need to provide two parameters:
- config file
- environment name

All other values will be resolved automatically.

### Sample command

```bash
uv run mcon_ragas_runner.py run --config=configs/ragas_tests/owui-sample.yaml --env=mdev
```

The script automatically:
* Resolves required placeholders (e.g., env)
* Fetches domain information from AWS SSM
* Applies {{domain_name}} replacement dynamically

### Commands

#### `run` — run the test suite

```text
  --config CONFIG             Path to config file (e.g., configs/ragas_tests/owui-bmw-general.yaml)
  --env ENV                   Environment name (e.g., mdev, mqa, mtmp)
  --output_folder             Path to output folder (e.g., test_runs)
  --aws_profile_name          AWS credentials profile name
  --comments COMMENTS         Comments for the test run
  --questions QUESTIONS       Comma-separated question IDs to run, supports ranges (e.g. B01,B02,B04-10).
                              Runs all questions if not specified.
  --tags TAGS                 Comma-separated tags to filter tests (e.g. smoke,regression).
                              Runs tests matching ANY tag. Runs all if not specified.
  --filter-mode {and,or}      How to combine --questions and --tags: 'or' runs tests matching either (default),
                              'and' requires both.
```

#### `list-tags` — inspect available tags

Prints all tags defined in the config with their associated test IDs, then exits. Does not require `--env`.

```bash
uv run mcon_ragas_runner.py list-tags --config=configs/ragas_tests/owui-sample.yaml
```

### Selecting specific tests to run

Use `--questions` to filter by ID, `--tags` to filter by tag, or combine both with `--filter-mode`.

```bash
# Single question
uv run mcon_ragas_runner.py run --config=configs/ragas_tests/owui-bmw-general.yaml --env=mdev --questions=B01

# Specific list
uv run mcon_ragas_runner.py run --config=configs/ragas_tests/owui-bmw-general.yaml --env=mdev --questions=B01,B03,B07

# Range (expands to B04, B05, B06, B07, B08, B09, B10)
uv run mcon_ragas_runner.py run --config=configs/ragas_tests/owui-bmw-general.yaml --env=mdev --questions=B04-10

# By tag
uv run mcon_ragas_runner.py run --config=configs/ragas_tests/owui-bmw-general.yaml --env=mdev --tags=smoke

# Questions AND tags (test must match both)
uv run mcon_ragas_runner.py run --config=configs/ragas_tests/owui-bmw-general.yaml --env=mdev --questions=B01-05 --tags=smoke --filter-mode=and
```

### `list-tags` — inspect tags in a config

Prints all tags defined in the config file alongside the IDs of tests that carry each tag, then exits without running any tests.

```bash
uv run ragas_runner.py list-tags --config=configs/agent-sample.yaml
```

Parameters:

```text
  --config CONFIG       Path to config file (e.g., configs/generic-sample.yaml)
```

## Test Results

### YAML file export:

```yaml
comments: my sample run
execution_timestamp: '20251010_173303'
openwebui_endpoint: https://dev.chat.soname.solutions
tests_results:
- actual_answer: 'The main ingredients of sushi are:
    ...
    fish, and properly prepared sushi rice with the right texture and seasoning.'
  evaluator: NovaPro-ScoreOutOfTen
  id: S01
  model_name: General AIAssistant
  question: What are main ingredients of sushi?
  reference_answer: Response must include mentions of Rice, Fish or seafood and Nori
  response_time_sec: 6.0
  score: 10.0
...

```

- Test results file is created for each run.
- Results are stored in OUTPUT_FOLDER (from command line parameter)
- Filename: "run_{test_suite_name}_{execution datetime}.yaml
- For each test the following metrics are stored:
  - `score`
  - `response_time_sec`

### Excel Export

In addition to the YAML result file, each test run is also exported to an Excel spreadsheet for easier analysis and sharing.

The Excel export reads the generated YAML result file and produces a formatted `.xlsx` report containing all executed tests and their metrics.

**Key features:**

- **Structured test results** – Each test appears as a row containing:
  - `id`
  - `question`
  - `reference_answer`
  - `actual_answer` 
  - `session_id` – session identifier for the request (populated for `BedrockAgentCoreRagasTestExecutor` runs; can be used to look up the request in CloudWatch logs)
  - `score`
  - performance metrics (`till_first_token_time_sec`, `response_time_wo_tftt_sec`, `response_time_sec`)
- **Automatic summary rows** – The report includes **Average** and **Total** rows for the metrics.
- **Conditional formatting** – Performance metrics and scores are color-coded to quickly highlight good, neutral, and problematic results.
- **Filtering support** – Excel auto-filter is enabled for easy sorting, filtering, and further analysis.

The file is saved in the same `OUTPUT_FOLDER` as the YAML test results.  
The output filename follows the naming convention:

`run_{test_suite_name}{execution_timestamp}{env}.xlsx`

Recommendations:

- Default output folder contains .gitignore file (so test results are not committed to git)
- If you want to save results (both yaml and excel, e.g. for future use as baseline) - move them to `saved_runs` folder
