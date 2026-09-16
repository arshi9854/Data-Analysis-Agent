# Data Analysis Agent

Explore CSV data with natural-language questions, generated Python analysis,
and charts. Built with Streamlit, pandas, matplotlib, and NVIDIA's
Llama-3.1-Nemotron-Ultra-253B-v1 model.

```text
    YOUR CSV          YOUR QUESTION              YOUR ANALYSIS
  +-----------+     +------------------+       +------------------+
  | rows and  | --> | "Plot sales by   | ----> | Explanation      |
  | columns   |     |  region"        |       | Chart            |
  +-----------+     +------------------+       | Inspectable code |
                                              +------------------+
```

## Features

- Upload a CSV and preview its first five rows.
- Get a dataset summary and suggested analysis questions on upload.
- Ask questions in plain English and generate pandas calculations.
- Request visualizations generated with matplotlib.
- Read a short explanation and expand the generated code for inspection.
- Revisit messages and charts within the current Streamlit session.

The application is a fixed pipeline of Python functions. Functions named
"agents" handle individual stages; there is no autonomous planning or
code-repair loop.

## Architecture

```text
  Browser
  +------------------------------------------------------------+
  | Streamlit: CSV upload | Dataset preview | Chat | Charts      |
  +------------------------------+-----------------------------+
                                 |
                                 v
  Streamlit server process
  +------------------------------------------------------------+
  | main()                                                     |
  |   |                                                        |
  |   +--> pandas DataFrame <--> st.session_state               |
  |   |                                                        |
  |   +--> DataInsightAgent       : upload-time summary         |
  |   +--> CodeGenerationAgent    : routing + Python generation |
  |   +--> ExecutionAgent         : execute against DataFrame   |
  |   +--> ReasoningAgent         : explain execution output    |
  +------------------------------+-----------------------------+
                                 |
                     Remote model calls over HTTPS
                                 |
                                 v
  +------------------------------------------------------------+
  | NVIDIA API: https://integrate.api.nvidia.com/v1              |
  | Model: nvidia/llama-3.1-nemotron-ultra-253b-v1                |
  | Client: OpenAI Python SDK                                   |
  +------------------------------------------------------------+
```

The full DataFrame is held in the application process. Model requests send
dataset metadata, user questions, and result excerpts as described below.
The application has no database, vector store, or persistent chat storage.

## Quick start

### 1. Install dependencies

Use a Python environment compatible with the versions in
[requirements.txt](requirements.txt), such as Python 3.11.

```bash
git clone https://github.com/arshi9854/Data-Analysis-Agent.git
cd Data-Analysis-Agent
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

On Windows PowerShell, activate with `.venv\Scripts\Activate.ps1` instead.

### 2. Configure NVIDIA API access

You need an NVIDIA API key with access to the model configured in
[data_analysis_agent.py](data_analysis_agent.py), plus network access to the
NVIDIA endpoint.

**The current source assigns the API key directly.** Before running, replace
that `api_key = ...` assignment in the configuration section with:

```python
api_key = os.environ["NVIDIA_API_KEY"]
```

The file already imports `os`. Set `NVIDIA_API_KEY` in your environment before
launching the app. For example, in macOS/Linux Bash or Zsh, this reads the key
without displaying it or placing its value in shell history:

```bash
printf 'NVIDIA API key: '
read -r -s NVIDIA_API_KEY
printf '\n'
export NVIDIA_API_KEY
```

Setting the environment variable alone will not work until the assignment is
updated. Do not reuse the credential committed in the source; its owner should
revoke or rotate it if active. Keep replacement credentials out of Git.

### 3. Launch the application

```bash
python -m streamlit run data_analysis_agent.py
```

Open the local URL printed by Streamlit, upload a CSV, and ask a self-contained
question. Start with synthetic data and run only in a disposable environment
without sensitive files or credentials beyond the required API key: generated
Python is executed directly, without a security sandbox.

## Try an example

Save this as `sales.csv` and upload it:

```csv
region,product,units,revenue
North,Notebook,10,50
South,Notebook,8,40
North,Pen,20,30
West,Notebook,6,30
South,Pen,12,18
West,Pen,10,15
```

Example questions:

- "What is the total revenue?"
- "What is the total revenue for each region?"
- "Plot total revenue by region as a bar chart."
- "Which product sold the most units?"

For manual checking, total revenue is **183**, regional revenue is **North: 80,
South: 58, West: 45**, and Pen has the most units at **42**. These are expected
values for this sample, not a claim of automated evaluation.

For the regional revenue question, an illustrative generated calculation is:

```python
result = df.groupby("region")["revenue"].sum()
```

Model-generated code may vary. Review results against known calculations.

## How a request moves through the application

### Upload flow

```text
  CSV upload
      |
      v
  pd.read_csv(file) --> Store DataFrame and filename in session
      |
      +--> df.head() --> Dataset preview
      |
      v
  DataFrameSummaryTool
  [row count, column names, types, missing-value counts]
      |
      v
  DataInsightAgent --> NVIDIA model
      |
      v
  Dataset description + 3-4 suggested questions
```

The summary prompt contains metadata rather than sample rows. A newly detected
filename resets chat messages and triggers a fresh summary.

### Question flow

```text
  User question + current DataFrame
                    |
                    v
       QueryUnderstandingTool [model call 1]
             "Does this need a chart?"
                    |
          +---------+---------+
          |                   |
        false                true
          |                   |
          v                   v
  CodeWritingTool    PlotCodeGeneratorTool
  pandas prompt      pandas + matplotlib prompt
          |                   |
          +---------+---------+
                    |
                    v
       CodeGenerationAgent [model call 2]
       Input: question + column names
                    |
                    v
       extract_first_code_block()
                    |
                    v
       ExecutionAgent: exec(code, {}, env)
       env: df, pd; plus plt and io for charts
                    |
                    v
       result object OR execution-error string
                    |
          +---------+------------------------+
          |                                  |
          v                                  v
  ReasoningCurator                    If Figure or Axes:
  Prepare result description          save chart in session
          |                                  |
          v                                  |
  ReasoningAgent [model call 3]               |
  Generate explanation                       |
          |                                  |
          +----------------+-----------------+
                           |
                           v
              Save assistant message; rerun UI
              Explanation + code + optional chart
```

Code-generation prompts require a fenced Python block and an output variable
named `result`. Plot prompts request one chart with a `(6, 4)` figure size and
titles/labels. These are prompt instructions, not enforced constraints.

`ExecutionAgent` catches Python exceptions and returns an error string. There
is no automatic correction or re-execution of failed code.

### Interface layout

```text
  +------------------------+-----------------------------------------+
  | Data Analysis Agent    | Chat with your data                     |
  |                        |                                         |
  | [Choose CSV]           | You: Plot total revenue by region       |
  |                        |                                         |
  | Dataset preview        | Assistant: Explanation                  |
  | First five rows        | [Expandable reasoning, when available]  |
  |                        | [View code]                             |
  | Dataset Insights       |                                         |
  | Summary and suggested  |          [Generated chart]              |
  | questions              |                                         |
  |                        | [Ask about your data...]                |
  +------------------------+-----------------------------------------+
           ~30%                              ~70%
```

## Model calls and data sent

All stages use the same configured NVIDIA model through the OpenAI-compatible
client. The SDK name does not mean requests are sent to OpenAI's endpoint.

| Stage | Input sent to model | Temperature | Max output tokens |
| --- | --- | --- | --- |
| Dataset insights | Dimensions, columns, data types, missing counts | 0.2 | 512 |
| Query classification | Current question | 0.1 | 5 |
| Code generation | Current question and column names | 0.2 | 1,024 |
| Explanation | Question and result excerpt, plot description, or error | 0.2 | 1,024 |

A successful dataset load followed by `N` questions makes `1 + 3N` logical
model calls. Each question's calls run sequentially. The explanation request
uses streaming; the final explanation is assembled before the chat redraws.

For non-chart results, the explanation receives only the first 300 characters
of the string representation. For charts, it receives a title/placeholder,
not the image or plotted values. Dataset-derived information can therefore
leave the application, and explanations have limited context.

## Code map

```text
  Data-Analysis-Agent/
  |-- data_analysis_agent.py   # UI, prompts, execution, model calls
  |-- requirements.txt        # Dependency minimum versions
  |-- README.md               # Setup and implementation guide
  `-- LICENSE                 # Apache License 2.0
```

| Function | Responsibility |
| --- | --- |
| `main` | Uploads, session state, chat orchestration, and rendering |
| `DataFrameSummaryTool` | Builds a prompt from DataFrame metadata |
| `DataInsightAgent` | Requests the upload-time summary |
| `QueryUnderstandingTool` | Classifies visualization intent |
| `CodeWritingTool` | Builds the pandas-only prompt |
| `PlotCodeGeneratorTool` | Builds the pandas/matplotlib prompt |
| `CodeGenerationAgent` | Routes the request and generates code |
| `extract_first_code_block` | Extracts the first fenced Python block |
| `ExecutionAgent` | Runs code and retrieves `result` |
| `ReasoningCurator` | Converts the result into an explanation prompt |
| `ReasoningAgent` | Streams the model response and separates explanation text |

Session state stores `df`, `current_file`, `insights`, `messages`, and `plots`.
Chat history is displayed but is not passed into subsequent model requests.

## Current limitations

- **Execution safety:** `exec()` runs model-generated Python in the application
  process. There is no sandbox, resource limit, or code approval step. Generated
  code can also mutate the shared DataFrame.
- **Answer validation:** successful execution does not establish analytical
  correctness. There are no automated reference checks or repair loops.
- **Limited generation context:** code generation sees column names, but no
  data types, sample values, or previous messages. Ask complete questions.
- **Limited explanations:** table output is truncated before explanation;
  chart explanations receive no chart image or underlying values.
- **Result rendering:** charts are rendered, but returned DataFrames/Series
  are not displayed as dedicated result tables. The calculated result header
  is currently unused; users primarily see explanation text and code.
- **Upload detection:** replacement files with the same filename may reuse the
  previous DataFrame. Use a different filename or start a new session.
- **Error handling:** summary and execution exceptions are caught; CSV parsing
  and other model calls lack equivalent application-level handling. Missing
  code fences can produce empty code and a `None` result.
- **Session lifecycle:** state is in memory, and stored figures are not cleared
  when datasets change. There is no persistent history or figure cleanup.
- **Deployment:** authentication, authorization, and operational monitoring are
  not implemented in this repository.

## Possible improvements

These are future extensions, not existing features:

- Isolate generated-code execution and impose time/memory limits.
- Add reference-answer evaluations and structured error handling.
- Include appropriate schema/value context and clarify ambiguous questions.
- Render result tables directly and ground chart explanations in plotted data.
- Detect uploads by content and clean up figures when replacing a dataset.
- Add secret management, access controls, and deployment observability.

## License

Licensed under the [Apache License 2.0](LICENSE). Preserve the existing source
copyright and license notices when redistributing or adapting the code.
