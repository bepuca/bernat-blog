---
title: "Building an Evaluation Harness for VSCode Copilot Chat"
subtitle: Because your prompts deserve better than vibe-checks
date: "2025-06-30"
image: chat_eval_thumbnail.png
---

:::{.post-thumbnail}
![](chat_eval_thumbnail.png)
:::

You built a **VSCode Copilot Chat prompt** that extracts key information from server error logs. You paste a log entry, it returns structured data: error type, severity, and affected component. It works great for some logs, but fails on others. You tweak the prompt to fix the failing cases - now the ones that worked are broken. **Without systematic testing, you're playing whack-a-mole**: every fix introduces new problems, and you can't tell if you're making progress or just moving issues around. Sound familiar?

While [VSCode Copilot Chat](https://code.visualstudio.com/docs/copilot/overview) is marketed for code interaction, it turns out that with [Agent mode](https://code.visualstudio.com/docs/copilot/chat/chat-agent-mode), [custom tools](https://code.visualstudio.com/docs/copilot/chat/chat-agent-mode#_agent-mode-tools), and [reusable prompts](https://code.visualstudio.com/docs/copilot/copilot-customization), **you can build agentic workflows with almost no code**. The batteries are included - file operations, web access, and [MCP](https://modelcontextprotocol.io/introduction) tools - making it well-suited for rapid prototyping of AI workflows that may or may not interact with your codebase. But there's a catch: while you can test your custom MCP servers and tools, **there's no way to evaluate the prompts that orchestrate them**. You're stuck with "it seems to work fine" - yet [evals are essential](https://hamel.dev/blog/posts/evals/) for [systematically improving AI solutions](https://www.bepuca.dev/posts/hdd-for-ai/).

This is a real problem. Without evaluation, you can't measure performance, catch regressions, or improve reliably. You are left between vibe checks and building a fully-fledged custom solution that could take weeks - effort that could be completely wasted if the workflow turns out to be less useful than expected. In this post, **we'll build a scrappy evaluation harness for VSCode Copilot Chat prompts** that hopefully gets us 80% of the value with 20% of the effort. The code is available at [github.com/bepuca/copilot-chat-eval](https://github.com/bepuca/copilot-chat-eval).

## The Problem

To build an evaluation harness for [VSCode Copilot Chat prompts](https://code.visualstudio.com/api/extension-guides/language-model), we need to solve a specific challenge: **how do we programmatically run the same prompt with different inputs and capture the results?**

Let's be concrete. Say you have a prompt that parses server logs. You want to:

1. Feed it 20 different log entries from your test dataset
2. Capture each extraction response
3. Check if those extraction are correct
4. Track performance over time

But VSCode Chat is designed for interactive use, not automation. There's no obvious API to send a message and get a response programmatically. And here's the crucial constraint: LLMs are sensitive to context, so our **evaluation must mimic manual use exactly**. If we test with a different context or invocation method, we're not actually testing what users experience - and we don't know what context differences might exist.

For simplicity, **we'll focus on single-turn interactions**: one prompt, one response. Multi-turn conversations add complexity we could tackle later.

Our requirements are straightforward:
- Define a reusable prompt that accepts parameters
- Create a dataset of test inputs
- Run each input through Copilot Chat automatically.
- Capture and save the responses for analysis

If we can do this, we can finally measure our prompts' performance.

## Exploring Our Options

Since VSCode doesn't expose chat functionality through an API, we need to get creative. The primary way to programmatically interact with VSCode is by [building an extension](https://code.visualstudio.com/api/get-started/your-first-extension). After digging through the documentation, I found three potential approaches:

1. **Build a [Chat Extension](https://code.visualstudio.com/api/extension-guides/chat)**: Create a chat participant that users invoke with `@participant`. This won't work because:
   - It changes the user's workflow (they'd have to type `@eval /myprompt` instead of just `/myprompt`).
   - Not supported in Agent mode (at least for now)
   - Different invocation = different LLM context = invalid evaluation. Off the table, then.
2. **Use the [Language Model API](https://code.visualstudio.com/api/extension-guides/language-model)**: Call Copilot's models directly from our extension. Promising, but:
   - No custom system prompts allowed (might not match chat's behavior).
   - Unclear if Agent mode and tools work through this API.
   - If we can't replicate chat features, we're not really testing the same thing.
3. **Automate VSCode commands**: Use the same commands that keybindings trigger to programmatically control the chat. A bit of a workaround, but:
   - Could reproduce exact user interactions.
   - Relies on somewhat undocumented behavior that might break.
   - No guarantees it'll work or keep working.

**Each of these options has significant drawbacks** that make evaluation difficult.

With [VSCode Chat Copilot going open source](https://code.visualstudio.com/blogs/2025/05/19/openSourceAIEditor), the community might eventually build proper evaluation tools. But that could take months - and often [something shipped today is better than perfection later](https://lucumr.pocoo.org/2025/2/20/ugly-code/).

Let’s take a practical approach and focus on what’s realistically achievable.

## Testing the Language Model API

Option 2 (Language Model API) seems most promising - if it works, we get clean programmatic access. But there's a critical question: does it use the same system prompt as the chat interface? If not, we're evaluating something different from what users experience.

To find out, **we need to see what prompts VSCode actually sends**. Following [Hamel's approach](https://hamel.dev/blog/posts/prompt/#setting-up-mitmproxy), we'll use [mitmproxy](https://mitmproxy.org/) to intercept VSCode's API calls and examine the system prompts.

### Setting Up mitmproxy

To set up mitmproxy to intercept VSCode's API calls:
1. Install: `uv tool install mitmproxy` or follow [official instructions](https://docs.mitmproxy.org/stable/overview/installation/)
2. Start the browser-based UI: `mitmweb`
3. Configure VSCode to use the proxy: Add `"http.proxy": "http://127.0.0.1:8080"` to `settings.json`. Or set up the "Http: Proxy" setting in the UI.
4. Trust the certificate so requests succeed: Follow [mitmproxy's instructions](https://docs.mitmproxy.org/stable/concepts/certificates/) for your OS.

### What Chat Sends

With mitmproxy running, we can trigger a chat message and searched for calls to `https://api.enterprise.githubcopilot.com/chat/completions`. Here's the request structure:
```json
{
    "messages": [
        {
            "role": "system",
            "content": "You are an AI programming assistant.\nWhen...",
            "copilot_cache_control": {
                "type": "ephemeral"
            }
        },
        {
            "role": "user",
            "content": "<environment_info>\nThe user's ...",
            "copilot_cache_control": {
                "type": "ephemeral"
            }
        },
        {
            "role": "user",
            "content": "<context>\nThe current date is...",
            "copilot_cache_control": {
                "type": "ephemeral"
            }
        }
    ],
    "model": "claude-sonnet-4",
    "temperature": 0,
    "top_p": 1,
    "max_tokens": 16000,
    "tools": [
	    // many and irrelevant at this point
    ]
}
```

The raw JSON has escaped characters, but when formatted for readability, the key parts are below. Note that the details of the Chat system prompt are not critical, we just need to see if it matches the Language Model API's.

::: {.callout-note collapse="true" icon=false}
## Chat System Prompt
```text
You are an AI programming assistant.
When asked for your name, you must respond with "GitHub Copilot".
Follow the user's requirements carefully & to the letter.
Follow Microsoft content policies.
Avoid content that violates copyrights.
If you are asked to generate content that is harmful, hateful, racist, sexist, lewd, or violent, only respond with "Sorry, I can't assist with that."
Keep your answers short and impersonal.
<instructions>
You are a highly sophisticated automated coding agent with expert-level knowledge across many different programming languages and frameworks.
The user will ask a question, or ask you to perform a task, and it may require lots of research to answer correctly. There is a selection of tools that let you perform actions or retrieve helpful context to answer the user's question.
You will be given some context and attachments along with the user prompt. You can use them if they are relevant to the task, and ignore them if not. Some attachments may be summarized. You can use the read_file tool to read more context, if needed.
If you can infer the project type (languages, frameworks, and libraries) from the user's query or the context that you have, make sure to keep them in mind when making changes.
If the user wants you to implement a feature and they have not specified the files to edit, first break down the user's request into smaller concepts and think about the kinds of files you need to grasp each concept.
If you aren't sure which tool is relevant, you can call multiple tools. You can call tools repeatedly to take actions or gather as much context as needed until you have completed the task fully. Don't give up unless you are sure the request cannot be fulfilled with the tools you have. It's YOUR RESPONSIBILITY to make sure that you have done all you can to collect necessary context.
When reading files, prefer reading large meaningful chunks rather than consecutive small sections to minimize tool calls and gain better context.
Don't make assumptions about the situation- gather context first, then perform the task or answer the question.
Think creatively and explore the workspace in order to make a complete fix.
Don't repeat yourself after a tool call, pick up where you left off.
NEVER print out a codeblock with file changes unless the user asked for it. Use the appropriate edit tool instead.
NEVER print out a codeblock with a terminal command to run unless the user asked for it. Use the run_in_terminal tool instead.
You don't need to read a file if it's already provided in context.
</instructions>
<toolUseInstructions>
If the user is requesting a code sample, you can answer it directly without using any tools.
When using a tool, follow the JSON schema very carefully and make sure to include ALL required properties.
Always output valid JSON when using a tool.
If a tool exists to do a task, use the tool instead of asking the user to manually take an action.
If you say that you will take an action, then go ahead and use the tool to do it. No need to ask permission.
Never use multi_tool_use.parallel or any tool that does not exist. Use tools using the proper procedure, DO NOT write out a JSON codeblock with the tool inputs.
NEVER say the name of a tool to a user. For example, instead of saying that you'll use the run_in_terminal tool, say "I'll run the command in a terminal".
If you think running multiple tools can answer the user's question, prefer calling them in parallel whenever possible, but do not call semantic_search in parallel.
When using the read_file tool, prefer reading a large section over calling the read_file tool many times in sequence. You can also think of all the pieces you may be interested in and read them in parallel. Read large enough context to ensure you get what you need.
If semantic_search returns the full contents of the text files in the workspace, you have all the workspace context.
You can use the grep_search to get an overview of a file by searching for a string within that one file, instead of using read_file many times.
If you don't know exactly the string or filename pattern you're looking for, use semantic_search to do a semantic search across the workspace.
Don't call the run_in_terminal tool multiple times in parallel. Instead, run one command and wait for the output before running the next command.
When invoking a tool that takes a file path, always use the absolute file path. If the file has a scheme like untitled: or vscode-userdata:, then use a URI with the scheme.
NEVER try to edit a file by running terminal commands unless the user specifically asks for it.
Tools can be disabled by the user. You may see tools used previously in the conversation that are not currently available. Be careful to only use the tools that are currently available to you.
</toolUseInstructions>
<editFileInstructions>
Don't try to edit an existing file without reading it first, so you can make changes properly.
Use the insert_edit_into_file tool to edit files. When editing files, group your changes by file.
NEVER show the changes to the user, just call the tool, and the edits will be applied and shown to the user.
NEVER print a codeblock that represents a change to a file, use insert_edit_into_file instead.
For each file, give a short description of what needs to be changed, then use the insert_edit_into_file tool. You can use any tool multiple times in a response, and you can keep writing text after using a tool.
Follow best practices when editing files. If a popular external library exists to solve a problem, use it and properly install the package e.g. creating a "requirements.txt".
If you're building a webapp from scratch, give it a beautiful and modern UI.
After editing a file, any new errors in the file will be in the tool result. Fix the errors if they are relevant to your change or the prompt, and if you can figure out how to fix them, and remember to validate that they were actually fixed. Do not loop more than 3 times attempting to fix errors in the same file. If the third try fails, you should stop and ask the user what to do next.
The insert_edit_into_file tool is very smart and can understand how to apply your edits to the user's files, you just need to provide minimal hints.
When you use the insert_edit_into_file tool, avoid repeating existing code, instead use comments to represent regions of unchanged code. The tool prefers that you are as concise as possible. For example:
// ...existing code...
changed code
// ...existing code...
changed code
// ...existing code...

Here is an example of how you should format an edit to an existing Person class:
class Person {
        // ...existing code...
        age: number;
        // ...existing code...
        getAge() {
                return this.age;
        }
}
</editFileInstructions>
<notebookInstructions>
To edit notebook files in the workspace, you can use the edit_notebook_file tool.Use the run_notebook_cell tool instead of executing Jupyter related commands in the Terminal, such as `jupyter notebook`, `jupyter lab`, `install jupyter` or the like.
Use the copilot_getNotebookSummary tool to get the summary of the notebook (this includes the list or all cells along with the Cell Id, Cell type and Cell Language, execution details and mime types of the outputs, if any).
</notebookInstructions>
<outputFormatting>
Use proper Markdown formatting in your answers. When referring to a filename or symbol in the user's workspace, wrap it in backticks.
<example>
The class `Person` is in `src/models/person.ts`.
</example>

</outputFormatting>
```
:::

::: {.callout-tip collapse="true" icon=false}
## First User Message
```text
<environment_info>
The user's current OS is: macOS
The user's default shell is: "zsh". When you generate terminal commands, please generate them correctly for this shell.
</environment_info>
<workspace_info>
The following tasks can be executed using the run_vs_code_task tool:
<workspaceFolder path="path/to/workspace">
<task id="shell: echo">
{
        "label": "echo",
        "type": "shell",
        "command": "echo Hello"
}
</task>

</workspaceFolder>
I am working in a workspace with the following folders:
- path/to/workspace
I am working in a workspace that has the following structure:

'''
- file1
- file2
'''

This is the state of the context at this point in the conversation. The view of the workspace structure may be truncated. You can use tools to collect more context if needed.
</workspace_info>
```
:::

::: {.callout-tip collapse="true" icon=false}
## Second User Message
```text
<context>
The current date is 24 June 2025.
</context>
<reminder>
When using the insert_edit_into_file tool, avoid repeating existing code, instead use a line comment with \`...existing code...\` to represent regions of unchanged code.
When using the replace_string_in_file tool, include 3-5 lines of unchanged code before and after the string you want to replace, to make it unambiguous which part of the file should be edited.
</reminder>
<userRequest>
What is the meaning of life?
</userRequest>
```
:::

Key insights for evaluation:

- **Temperature is 0** - good for reproducibility
- **Environment info is injected** - OS details make context vary between users
- **Date is injected** - same prompt on different days = different context

Perfect reproducibility is impossible since VSCode injects dynamic context. But that's OK - we're building a practical tool, not a perfect one. **Some signal beats no signal**.

### What the Language Model API sends

To see if the Language Model API uses the same system prompt as chat, I built a minimal VSCode extension that calls the API directly. The key part:


```typescript
const [model] = await vscode.lm.selectChatModels({
    vendor: 'copilot',
    family: 'claude-sonnet-4'
});
const messages = [
    vscode.LanguageModelChatMessage.User('What is the meaning of life?')
];
const request = model.sendRequest(messages, {}, token);
```

When I ran this extension and checked mitmproxy, here's what the Language Model API sends:

```json
{
    "messages": [
        {
            "role": "system",
            "content": "Follow Microsoft content policies...",
        },
        {
            "role": "user",
            "content": "What is the meaning of life?"
        }
    ],
    "model": "claude-sonnet-4",
    "temperature": 0.1,
    "top_p": 1,
    "max_tokens": 16000,
    "n": 1,
    "stream": true
}
```

Once again, formatting the system prompt for readability:

::: {.callout-note collapse="true" icon=false}
## Language Model API System Prompt
```text
Follow Microsoft content policies.
Avoid content that violates copyrights.
If you are asked to generate content that is harmful, hateful, racist, sexist, lewd, or violent, only respond with "Sorry, I can't assist with that."
Keep your answers short and impersonal.
Use Markdown formatting in your answers.
Make sure to include the programming language name at the start of the Markdown code blocks.
Avoid wrapping the whole response in triple backticks.
The user works in an IDE called Visual Studio Code which has a concept for editors with open files, integrated unit test support, an output pane that shows the output of running the code as well as an integrated terminal.
The active document is the source code the user is looking at right now.
You can only give one reply for each conversation turn.
```
:::

**The verdict: completely different system prompts.**

Chat gets a comprehensive system prompt with detailed instructions about tools, file editing, and workspace context. The Language Model API gets a minimal prompt focused on basic content policies.

Key differences:

- **System prompt**: Chat has ~200 lines of instructions, API has ~10 lines
- **Context injection**: Chat auto-adds environment info, API gives you full control
- **Temperature**: Chat uses 0, API defaults to 0.1

**Bottom line:** The Language Model API evaluates a different system than what users experience in chat. Our best remaining option now is option 3 - automating VSCode commands to control the actual chat interface.

## Hacking VSCode Commands

Since the Language Model API won't work, I needed to find a way to control VSCode's chat interface programmatically. Most VSCode actions can be triggered via [commands](https://code.visualstudio.com/api/references/commands). While not all of them are explicitly documented, the rest can be found in [`keybindings.json`](https://code.visualstudio.com/api/references/commands#simple-commands).

I extracted all chat commands from VSCode's keybindings with a simple grep:

```bash
grep -o 'workbench\.action\.chat\.[^"]*' keybindings.json | sort -u
```

This found 50+ commands. After some experimentation, the key ones for automation:

- `workbench.action.chat.newChat` - Start fresh conversation
- `workbench.action.chat.attachFile` - Attach prompt file
- `workbench.action.chat.open` - Focus and send message to chat
- `workbench.action.chat.export` - Export chat to JSON

### Building the Evaluation Loop

The automation flow is straightforward. For each record in the evaluation dataset:

1. Start a new chat
2. Attach the prompt file
3. Send the test input
4. Wait for completion
5. Export and save results

Here's the core loop:

```typescript
const promptUri = vscode.window.activeTextEditor?.document.uri;
const resultsFile = await initResultsFile(root, promptUri);
for (const rec of records) {
    if (typeof rec.input !== 'string') continue;

    await vscode.commands.executeCommand('workbench.action.chat.newChat');
    await vscode.commands.executeCommand('workbench.action.chat.attachFile', promptUri);
    await vscode.commands.executeCommand('workbench.action.chat.open', rec.input);

    await sleep(rec.waitMs);  // let the chat finish, as we cannot query the status
    await vscode.commands.executeCommand('workbench.action.chat.export');
    await sleep(500);  // give VS Code time to write the file

    const exportDir = path.dirname(promptUri.fsPath); // default destination
    await collectAndAppendChatExport(exportDir, resultsFile);
}
```

## Example: Country Capitals

Let's test this with a simple prompt that takes a country name and returns its capital:

`capital.prompt.md`
```markdown
---
mode: agent
tools: []
---
The user provides a country and you should answer with only the capital of that country.
```

The dataset format is simple - each test case needs:
- `input`: The message to send to the prompt
- `waitMs`: How long to wait for the response (we can't detect completion)

`dataset.json`
```json
[
    {"input": "France", "waitMs": 4000, "capital": "Paris"},
    {"input": "Japan", "waitMs": 4000, "capital": "Tokyo"},
    {"input": "Spain", "waitMs": 4000, "capital": "Madrid"}
]
```

Running the extension processes each test case and exports the results to `.github/evals/<prompt>/<datetime>.json`. Then we can evaluate the responses:

`eval_capital.py`
```python
import json
import sys
from pathlib import Path

dataset = json.loads(Path(sys.argv[1]).read_text())
results = json.loads(Path(sys.argv[2]).read_text())

correct = 0
for record, result_chat in zip(dataset, results):
    answer = result_chat["requests"][0]["response"][0]["value"]
    correct += 1 * (record["capital"] in answer)

accuracy = correct / len(dataset) * 100
print(f"Accuracy: {accuracy:.2f}% ({correct}/{len(dataset)})")
```

```text
$ python eval_capital.py dataset.json .github/evals/capital/20250625-0840.json
Accuracy: 100.00% (3/3)
```

## Seeing it in Action

![Demo of the evaluation harness](https://github.com/bepuca/copilot-chat-eval/blob/main/demo.gif?raw=true)

**The evaluation harness works!** We can now systematically test our prompts and track performance over time.

While our example uses simple string matching, you can make evaluation as sophisticated as needed - LLM-as-judge for complex outputs or multi-dimensional scoring, for instance. The key is starting simple and iterating.


## Conclusion

**Yes, we can evaluate VSCode Copilot Chat workflows** - with significant limitations. Our approach has **rough edges**:

- **Sequential execution** - Evaluations run one record at a time, no parallelization
- **Fixed wait times** - We must guess how long each prompt takes since there's no way to query completion status, leading to either wasted time or incomplete responses
- **Manual save dialog** - Users must press Enter for each evaluation run since the export command doesn't accept a file path

And it only works for a subset of prompts:

- **Read-only** - No side effects like file modifications or API calls. Running these in evaluation could cause real damage or spam external services.
- **Stateless** - Don't depend on current workspace changes or git state. Reproducing "review my current changes" would require setting up different workspace states for each test case.
- **Single-turn** - One input, one output. Multi-turn conversations require simulating user responses, which adds significant complexity.
- **Time-insensitive** - VSCode injects the current date into prompts. If your workflow depends on "today's date," results will vary between evaluation runs.

Despite these constraints, many useful workflows still fit within them: generating code snippets, writing documentation, analyzing error messages, or converting data formats. And crucially, having imperfect evaluation beats having none at all.

**Where this helps most: early experimentation.** When you're testing whether a prompt idea even works, this approach provides basic feedback without building a full custom solution.

The workflow becomes:

1. Build your prompt in VSCode using familiar tools.
2. Create a test dataset with representative inputs.
3. Build an evaluation script to derive metrics.
4. Run the evaluation harness to get systematic feedback.
5. Iterate based on concrete results rather than guesswork.
6. Make data-driven decisions about next steps.

This combination - fast workflow development in VSCode Copilot Chat + actual performance measurement - lets you quickly validate whether a workflow delivers real value and how reliable it is. With both pieces of data, you can make informed decisions about whether to invest in a custom solution.

### Further work

This extension is a proof of concept that works by bending VSCode to our will. It may be brittle and break as the platform evolves, especially at the current pace of AI tooling development. However, having a working blueprint makes iteration easier than starting from scratch.

With VSCode Chat going open source, there may be opportunities to build more robust evaluation tools with official support.

The capital cities example was deliberately simple. Agent mode prompts can reference tools via hashtags (e.g., `#search_repositories`, `#file_search`), and these tool-using, agentic workflows are where the possibilities expand significantly. The evaluation harness captures outputs regardless of which tools were invoked, making it just as applicable to complex workflows as simple ones.

## References

- [Repo with the code: bepuca/copilot-chat-eval](https://github.com/bepuca/copilot-chat-eval)
- [VSCode Copilot Chat](https://code.visualstudio.com/docs/copilot/overview)
- [Agent mode](https://code.visualstudio.com/docs/copilot/chat/chat-agent-mode)
- [Custom tools](https://code.visualstudio.com/docs/copilot/chat/chat-agent-mode#_agent-mode-tools)
- [Reusable prompts](https://code.visualstudio.com/docs/copilot/copilot-customization)
- [Model Context Protocol (MCP)](https://modelcontextprotocol.io/introduction)
- [Your AI product needs Evals - Hamel's Blog](https://hamel.dev/blog/posts/evals/)
- [Hypothesis Driven Development for AI Products](https://www.bepuca.dev/posts/hdd-for-ai/)
- [VSCode Language Model API](https://code.visualstudio.com/api/extension-guides/language-model)
- [Building a VSCode extension](https://code.visualstudio.com/api/get-started/your-first-extension)
- [Chat Extension API](https://code.visualstudio.com/api/extension-guides/chat)
- [VSCode Chat Copilot going open source](https://code.visualstudio.com/blogs/2025/05/19/openSourceAIEditor)
- [Ugly Code and Dumb Things - Armin Ronacher](https://lucumr.pocoo.org/2025/2/20/ugly-code/)
- [Fuck You, Show Me The Prompt - Hamel's Blog](https://hamel.dev/blog/posts/prompt/#setting-up-mitmproxy)
- [mitmproxy](https://mitmproxy.org/)
- [VSCode commands](https://code.visualstudio.com/api/references/commands)
- [Copilot Chat Eval Repository](https://github.com/bepuca/copilot-chat-eval)