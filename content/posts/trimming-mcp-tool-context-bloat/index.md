---
title: "Trimming MCP Tool Context Bloat"
date: 2026-10-23T00:00:00-07:00
draft: false
toc: true
tags: ["mcp", "llm", "ai", "homelab"]
---

One of the ways to close the performance gap between local LLMs and frontier models is to give local LLMs access to your private data that you either don't feel comfortable giving to commercial providers or where it would simply be impractical to do so. I'm talking about sources of data like your personal notes, your financial documents, your health records, your home automation systems, and similar data that can give LLMs the context they need to become truly useful in your day-to-day life.

One of the best ways to do this is to layer MCP (Model Context Protocol) services on top of these data sources. I run MCP servers for my Paperless, Bookstack, Joplin, Searxng, Home Assistant, Gitea, and my LLM persistent memory systems. With these MCP services, my local LLMs can quickly access specifics about my homelab or my home or autos or prescriptions or whatever else it needs to give me accurate advice or get the task at hand done.

These connections have proven invaluable for me, but they also created a new problem: context bloat.

The reason has to do with the way that MCP servers advertise their capabilities to LLMs. The primary way that an MCP server advertises its capabilities is through its `tools/list` endpoint, which is an enumeration of all of the operations the MCP server provides, including a description and the input arguments. For some MCP servers, this list can get quite lengthy - the [MCP server I use for persistent LLM memory](https://github.com/doobidoo/mcp-memory-service) advertises 26 operations. Bookstack MCP advertises 43! Each of these tool lists consumed 2.5-3.5k tokens. SearXNG's tool list consumed 1,700 tokens just to advertise four tools. 

Add on the lists for gitea, home assistant, and my homegrown paperless-rag MCP server, and just to start up a new chat session in my Open WebUI client my local LLM had to process well over 15,000 tokens! Not only did that take a noticeable amount of time, but when working with a 64k context window like I often do that's a huge chunk of your window gone right off the bat.

## Auditing your MCP Environment

Before you look for savings, it's a good idea to audit your MCP environment to identify the biggest potential wins. 

### Inspecting Tool Definitions

The official MCP Inspector can connect directly to any MCP server and show precisely what it advertises. For HTTP-based servers, you can use its CLI mode:

```bash
npx @modelcontextprotocol/inspector --cli \
  --server-url http://example.mcp.server/mcp \   # point it at each of your MCP servers in turn and save the responses
  --transport http \
  --method tools/list \
  --format json > mcp-tools.json
```

This saves the complete metadata to a file. Here's a snippet of gitea's tools/list that describes the create_branch tool.

```json
{
  "result": {
    "tools": [
      {
        "name": "create_branch",
        "description": "Create a new branch in a repository, optionally from a specific source branch (defaults to the repository's default branch).",
        "inputSchema": {
          "type": "object",
          "properties": {
            "branch": {
              "type": "string"
            },
            "old_branch": {
              "description": "source branch (default: repo default)",
              "type": "string"
            },
            "owner": {
              "description": "repo owner",
              "type": "string"
            },
            "repo": {
              "description": "repo name",
              "type": "string"
            }
          },
          "required": [
            "owner",
            "repo",
            "branch"
          ]
        },
        "annotations": {
          "title": "Create a new branch",
          "readOnlyHint": false,
          "idempotentHint": false
        }
      },
      [SNIP]
```

That's a relatively succinct tool description - they can get much longer than that!

Once you've saved the tools/list responses you can examine how much metadata each server publishes:

```bash
# Number of tools
$ jq '.result.tools | length' gitea-tools.json
61

# Tool definitions ranked by size in characters
$ jq -r '.result.tools[] | [.name, (tojson | length)] | @tsv' gitea-tools.json | sort -k2,2nr
pull_request_write      2368
create_or_update_file   1770
actions_run_read        1736
issue_write     1728
pull_request_read       1525
pull_request_review_write       1460
list_issues     1192
actions_run_write       1076
attachment_read 1026
label_write     1014
actions_config_write    1004
create_repo     934
notification_read       923
issue_read      901
package_read    896
project_write   895
list_pull_requests      826
actions_config_read     799
project_read    753
delete_file     739
list_commits    736
milestone_write 723
timetracking_read       720
search_repos    708
wiki_write      707
milestone_read  687
create_commit_status    681
label_read      679
get_repository_tree     655
timetracking_write      643
get_file_contents       640
create_release  632
search_issues   617
notification_write      608
list_releases   603
get_dir_contents        577
package_write   569
create_tag      567
create_branch   555
search_org_teams        545
compare_commits 542
fork_repo       517
get_commit      503
wiki_read       499
get_commit_status       496
list_tags       493
list_branches   492
create_organization     478
delete_release  461
delete_branch   460
delete_tag      455
list_org_repos  437
rename_branch   424
search_users    397
get_tag 392
list_my_repos   385
get_latest_release      379
get_release     354
get_user_orgs   346
get_gitea_mcp_server_version    287
get_me  196
```

The last command is particularly useful: it identifies the tools contributing most to the bloated specification, which will come in handy in a moment.

Gitea's tools/list is 45,000 characters long! Character-to-token conversion is an imperfect science, but that's likely to be something like 13k tokens if passed directly to an LLM.

SearXNG only has 4 tools, but even it would consume around 2,000 tokens.

```bash
searxng_web_search          641   3670
searxng_search_suggestions  132   536
searxng_instance_info       180   807
web_url_read                1202  1956
```

Once you have a better idea of what your MCP servers are advertising, you can start to think about how to get some context savings.

## Trimming Context Bloat - MCP Server Controls

The first opportunity to reduce the bloat is to have the server provide more succinct tool/lists to the MCP client.

### Lightweight Mode

It's not very common, but some servers actually offer a lightweight mode of their tools list. My SearXNG server (`ihor-sokoliuk/mcp-searxng`) supports a `SEARXNG_LITE_TOOLS` environment variable that reduces tool schemas to only the essential arguments. Simply by adding this to my docker compose file:

```yaml
environment:
  SEARXNG_LITE_TOOLS: "true"
```

tells the server to advertise only the essential information about its tools. After enabling:

```bash
searxng_web_search          43    213
searxng_search_suggestions  38    216
searxng_instance_info       39    150
web_url_read                429   589
```

Total: 1,168 characters. An 83.2% reduction without removing any tools. Now, this may not be a magic bullet - less information about how to use the service may mean that the LLM is less effective using it. You need to test to ensure that your LLM can still use your MCP service with high quality.

### Edit the Source

If you control the MCP server code, you can shorten responses directly. A verbose tool description like "This tool allows the assistant to search through the user's stored memory database using a natural-language semantic similarity query. The tool should be used whenever the user asks about previous conversations..." can be reduced to "Search persistent memories by semantic similarity. Use for past conversations, preferences, and decisions." For a server with dozens of tools, this saves substantial space. If you've had AI build your MCP server, double-check what it produced for the tools list!

## Trimming Context Bloat - MCP Client Controls

A far more common way to reduce MCP context bloat is to limit what the MCP Client sends to the LLM.

### Tool Filter List

This is the most direct tool in your arsenal for managing MCP context bloat.

Many clients have the ability to filter the tool list so that you only expose the operations you actually need the LLM to use. Often, you'll see many administrative and other esoteric operations in a tools/list that aren't useful in day to day operation. Or sometimes you only want read access to an MCP server so ideally you wouldn't send write-related operation info at all (aside: don't mistake this for actual security - you still need to enforce read-only access through other mechanisms).

I'll refer to Open WebUI as my example client throughout this post, but there are similar features in many other clients. For Open WebUI, **Function Name Filter List** under Settings → Admin → Integrations lets you restrict which functions are exposed to the model for any given MCP connection.

Considering Gitea MCP server, some of the worst offender tools were things you might not even need, such as actions_run_read (if we're not using Gitea actions) or issue_write (if we're not using Gitea's issue tracking). If your Gitea MCP server offers 40 operations but your workflows only need six, exposing just those six reduces the function definitions sent to the model dramatically. For my own Open WebUI instance I only reference Gitea for knowledge, not to actually create or edit code, so I filter Gitea MCP down to `list_my_repos, search_repos, get_repository_tree, get_dir_contents, get_file_contents, list_branches, list_commits, get_commit, compare_commits, actions_run_read`. My coding clients like Opencode use `git` operations directly, so I never use the write-related operations on Gitea MCP.

### Lazy Tool Search

Another technique clients sometimes provide is what I call lazy tool search - the LLM does its own discovery of available tools only when it needs them based on the context of the conversation. In Open WebUI, the **Tool Search** feature keeps large tool definitions out of every model request until they're needed. Enable it under Settings → Admin → Interface, and configure the deferral threshold - e.g. if you set the threshold to 400, Open WebUI will not forward any tool definitions larger than 400 characters to the LLM at chat initialization time. This requires configuring the LLM with Native function-calling mode so it can do the auto-discovery appropriately.

Although this can lead to enormous start-of-chat context savings, it also carries a lot of risk - now that you're not explicitly telling the LLM what tools are available, you're relying on the LLM to proactively search for and find them, which is a much bigger ask. Make sure your LLM is strong with tool calling, and maybe give it some initial hints in the system prompt to give it a head's up.

## Measuring the Actual Impact

It's best to iterate through these strategies and measure your savings AND the resulting LLM respose quality after each adjustment so you can find your biggest wins. One important caveat: the JSON returned by `tools/list` is not necessarily identical to what the model receives. Your MCP Client can rename tools, filter them, and translate their schemas into the LLM provider's function-calling format before forwarding information to the LLM.

Because I'm running llama.cpp behind llama-swap, I can measure the actual overhead wins precisely. I developed the following testing methodology:

1. Disable MCP tools, start a new chat in Open WebUI (beware of caching here - I got really confused until I realized Open WebUI wasn't picking up new settings right away), and record `prompt_tokens` from the OpenAI-compatible API usage data
2. Enable your MCP tools and repeat the same request in a fresh conversation
3. Compare `prompt_tokens`
4. Repeat with Tool Search enabled

Use fresh conversations with identical messages so that chat history and prompt-cache behavior don't distort the comparison.

Measuring quality preservation is more of a subjective pursuit, but it's usually fairly obvious when a tweak has tanked quality if you use a few consistent queries to test out the service before and after a change.

## Conclusion

By applying a combination of these techniques I got my initial chat start up context down to about 3k tokens. The improvement in startup time is dramatic, and I can have longer conversations without exhausting my context window now. I may back off a bit if I feel that one or more of my MCP Servers isn't being used as effectively, but so far it seems like I've managed to preserve most quality while achieving greater than 80% MCP startup token savings. For a local LLM, every token saved counts, so I'm quite happy with these results. I hope you find similar wins!