# Set up a working directory

For the purposes of this walkthrough, let's assume all files are in a directory called `genai-tutorial`:

```
cd ~
mkdir genai-tutorial
cd genai-tutorial
```

# Set up `fetcher` and download some content

[Fetcher](https://github.com/odewahn/fetcher) is a tool for downloading content from the O'Reilly learning platform. You provide the content identifier you want to download. Fetcher creates a directory based on the identfier and work title, and then downloads the source text into it. You need a JWT to be able to get full content. You can see the additional capabilities of fetcher at https://github.com/odewahn/fetcher.

To install it on OSX, doubleclick the fetcher package file from https://drive.google.com/drive/u/0/folders/15UZ9jfqb9bepiN4uNrSIZnJsiVXjNWX0.

## Startup

Open a terminal

```
cd ~/genai-tutorial
fetcher
```

It will take a minute for fetcher to start...

## Set up authentication

Run the following command once the REPL starts:

```
auth
```

You will be prompted to enter a JWT. If you don't have one, you can still use fetcher, but you will only get content previews. Note that this will create a configuration file called in your home directory called `.fetcher`. You should keep the content of this file private.

## Download content

You pull content using the `init` command:

```
init --identifier=9781492092384
```

Fetcher will create a direcory based on the identifier and title slug, and start downloading the content into it:

```
- 9781492092384-data-mesh
-- README.md
-- metadata.yml
-- source
   -- ...
   -- 00009-ch01-1-data-mesh-in-a-nutshell.html
   -- 00010-ch02-2-principle-of-domain-ownership.html
   -- 00011-ch03-3-principle-of-data-as-a-product.html
   -- 00012-ch04-4-principle-of-the-self-serve-data-platform.html
   -- ...
```

The `metadata.yml` file contains the a subset of the product metadata:

```
authors: Dehghani, Zhamak
content_format: book
description: <marketing description>
duration_seconds: null
format: book
identifier: '9781492092384'
issued: '2022-03-09'
publishers: O'Reilly Media, Inc.
title: Data Mesh
topics: Data Mesh
virtual_pages: 569
```

The `source` directory contains the text of each element in the project. Note that the file type depends on the type of content: books will have `html` data, while courses will be in `markdown` format.

# Prompt Templates

Once you have content downloaded, you can start defining the kinds of prompts you want to perform with it. Prompts are defined using the [jinja templating language](https://jinja.palletsprojects.com/en/3.1.x/templates/).

## Setup

You'll need a directory to store the prompt templates and automation scripts. Download this into the root your working directory:

```
cd ~/genai-tutorial
git clone https://github.com/odewahn/prompts
```

You will then have a directory that looks like this:

```
- prompts
  - tasks
    - extract-key-points.txt
    - ...
  - personas
    - oreilly-short.txt
  - scripts
    - summarizer.jinja
```

Here is a description of the contents for each folder:

## `tasks`

The `tasks` folder contains templates for the common tasks you want an LLM to perform. For example, things like extracting key points, writing a narrative summary, or creating a narrative summary.

For example, here is the contents of the `extract-key-points.txt` task:

```md
Here is a selection from a book called {{title}}:

---

{{block}}

---

Produce a bullet-point summary of the selection from {{title}}.
```

The task, as well as any metadata such as `{{title}}` and `{{block}}` elements is supplied to `prompter` as described below. The `{{block}}` element is a special, reserved word and is used to supply a block of text extracted from the content you downloaded from `fetcher`. In general, a chapter (much less an entire book) is far too long to fit within the context window of most models. `prompter` is a tool for helping manage this problem by giving you many ways to break contento into smaller blocks and do prompting with it.

## `personas`

The `personas` directory contains prompts related to the tone of voice and approach the LLM should use when it's performing the task. (This is sometimes also called the system prompt.) For example, you might want than LLM to sound like a helpful tutor, or perhaps a pirate.

Here is the sample of the `oreilly-short.txt` persona:

```md
Imagine you are an expert in a technical field, tasked with explaining a complex topic to a smart novice. Whether speaking or writing, your tone should be informal, helpful, and friendly, yet rigorously thorough. Emphasize clarity and engagement, ensuring that your explanation is accessible while maintaining depth. Your goal is to create a seamless experience where concepts take center stage, guiding the audience through the information with organized structure and eliminating unnecessary details. Consider the audience's potential prior knowledge and approach the explanation as if addressing an intelligent novice.
```

You supply the persona as an option to `prompter` as described below.

## `scripts`

prompter provides two ways to work with content: a REPL mode and a script mode. The section below will uses the REPL where you enter commands that are run one step at a time. However, much like a scripting language, you can also store commands in a file and then run them all at once. This enables you to scale the production of content once you've figured out how the original content should be formatted and chunked and decided on your prompts.

Here is a sample script that is included in the `scripts` direcory. Note that a script is also defined using a jinja template, so you can do some basic functions like branching and logic:

```bash
# This script assumes you used fetcher to download the content
# Before running this script, be sure to use the command
#    cd -dir=<content directory>
#
init
{% if format == "book" %}
   # Load a book
   load --fn=source/*.html
   transform --transformation="html2md,token-split"
   filter --where="block_tag like '%-ch01%' or block_tag like '%-ch02%'"
{% else %}
   # Loading {{format}}
   load --fn=source/*.md
   transform --transformation="token-split"
{% endif %}
# Extract the key points
prompt --task=../prompts/tasks/extract-key-points.txt --persona=../prompts/personas/oreilly-short.txt --global=metadata.yaml
squash --delimiter="\n************** SECTION BREAK *****************\n"
prompt --task=../prompts/tasks/cleanup-merged-blocks.txt --global=metadata.yaml
transfer-prompts --group_tag=cleaned-summaries
prompt --task=../prompts/tasks/convert-summary-to-narrative.txt --persona=../prompts/personas/oreilly-short.txt --global=metadata.yaml
transfer-prompts --group_tag=narrative-summary
dump --dir=.
prompt --task=../prompts/tasks/create-audiobook-narration.txt --persona=../prompts/personas/oreilly-short.txt --global=metadata.yaml
transfer-prompts
dump --dir=. --extension=audio-narration.txt
```

# Create summaries with prompter

[Prompter](https://github.com/odewahn/prompter) is a tool for automating the process of applying prompt templates to blocks of content. It provides a REPL that allows you to:

- Load content into a local SQLite database
- Convert the content into a format approriate for use with an LLM (e.g., epub => markdown)
- Break the content into smaller blocks that can fit within the context window of most LLMs (currently 8000 tokens). Most books or courses will be 200,000 tokens or more, so they require som degree of preprocessinf.
- Apply prompt templates (both task and persona) and metadata the blocks and sending them to the LLM for completion
- Store and manage the LLM output

Commands in prompter have a basic syntax that consists of a command name and a set of arguments that follow typical command line format. For example, completing a prompt looks like this:

```
prompt --task=../prompts/tasks/extract-key-points.txt --persona=../prompts/tasks/oreilly-short.txt --global=../metadata.yaml
```

You can find full documentation for prompter at https://github.com/odewahn/prompter.

The following sections assume you have followed the instructions above and downloaded some content and prompts into a root directory.

## Installation

Download and install prompter from https://drive.google.com/drive/u/0/folders/15UZ9jfqb9bepiN4uNrSIZnJsiVXjNWX0 and install it. Once this is complete, open VS Code on your root directory:

```
cd ~/genai-tutorial
code .
```

Within VSCode, open a terminal and type `prompter`. Your environment should look something like this:

![prompter in vscode](prompter-in-vscode.png)

## Navigating to the content repo

Once prompter starts, type `pwd` to see your current location in the filesystem:

```
prompter> pwd
/Users/odewahn/genai-tutorial
```

Use `ls` to see the contents of the directory:

```
prompter> ls
total 0
drwxr-xr-x  6 odewahn  staff  192 Jun 27 10:56 9781492092384-data-mesh
drwxr-xr-x  6 odewahn  staff  192 Jun 27 10:55 prompts
```

Use `cd --dir=<direcotory>` to change into the directory you downloaded with `fetcher`:

```
prompter> cd --dir=9781492092384-data-mesh
prompter> ls
total 24
-rw-------   1 odewahn  staff   121 Jun 27 10:56 README.md
-rw-------   1 odewahn  staff   285 Jun 27 10:56 init.promptlab
-rw-r--r--   1 odewahn  staff  1500 Jun 27 10:56 metadata.yaml
drwxr-xr-x  35 odewahn  staff  1120 Jun 27 10:58 source
```

# Run the script to produce the summaries

Run the script to create the summaries using this command:

```
run --fn=../prompts/scripts/summarizer.jinja
```

This will churn for a few minutes, but it's only doing two chapters, so it won't take too long. (A full book might take 15-20 minutes as prompter works now, although that time could be improved by paralellizing the requests).

Once it's complete, note the new markdown files in the root of the content directory. Also, note that you now what a file called `prompter.db` in your directory. This is a sqlite database that has all the blocks and content from the

# Create the summary in Atlas

- Create a new Atlas project at https://atlas.oreilly.com/
- Clone it locally
- Add the new file
- Configure the project
- Build

# Self-paced prompter demo

This section describes how to use prompter in more depth, and walks through many of the the commands in the main script one by one. The goal of this section is to explain the underlying concepts required to create new scripts for different types of works. For example, if you wanted to make a glossary for a book, study questions, or other types of content.

## Create a new database

Running this command will create a new database and back up the existing copy:

```
init
```

## Loading the content

Use the following command to load all the HTML files in the source directory:

```
load --fn=source/*.html
```

Here's a sample of the output:

```bash
prompter> load --fn=source/*.html
[17:03:22] Loading file source/*.html                                                                             main.py:417
    Loading file source/00000-cover-cover.html                                                             main.py:432
    Loading file source/00001-dedication01-praise-for-data-mesh.html                                       main.py:432
    Loading file source/00002-titlepage01-data-mesh.html                                                   main.py:432
    Loading file source/00003-copyright-page01-data-mesh.html                                              main.py:432
    Loading file source/00008-part01-i-what-is-data-mesh.html                                              main.py:432
    Loading file source/00009-ch01-1-data-mesh-in-a-nutshell.html                                          main.py:432
    Loading file source/00010-ch02-2-principle-of-domain-ownership.html                                    main.py:432
    ...
```

Run the `blocks` command to see blocks that were just created:

```
prompter> blocks
                                                       Current Blocks
┏━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━┓
┃ block_id ┃ block_tag              ┃ parent_block_id ┃ group_id ┃ group_tag        ┃ block                   ┃ token_count ┃
┡━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━┩
│ 1        │ 00000-cover-cover.html │ 0               │ 1        │ director-himself │ <div                    │ 7           │
│          │                        │                 │          │                  │ id="sbo-rt-content"><f… │             │
│          │                        │                 │          │                  │ data-ty                 │             │
│ 2        │ 00001-dedication01-pr… │ 1               │ 1        │ director-himself │ <div                    │ 593         │
│          │                        │                 │          │                  │ id="sbo-rt-content"><s… │             │
│          │                        │                 │          │                  │ class=                  │             │
|                      ..........                                                                                           |
│          │                        │                 │          │                  │ class=                  │             │
│ 32       │ 00031-colophon02-colo… │ 31              │ 1        │ director-himself │ <div                    │ 180         │
│          │                        │                 │          │                  │ id="sbo-rt-content"><s… │             │
│          │                        │                 │          │                  │ data-t                  │             │
└──────────┴────────────────────────┴─────────────────┴──────────┴──────────────────┴─────────────────────────┴─────────────┘

32 blocks with 131,892 tokens.
Current group id = (1, 'director-himself')

The following fields available in --where clause:
['block_id', 'block_tag', 'parent_block_id', 'group_id', 'group_tag', 'block', 'token_count']

```

Note that each block has a unique field called `block_tag` that is based off the original filename. You can use this name to refer to specific block or group of blocks.

You can view the contents of a block using the `dump` command with a `--where` clause to select the one you want to view:

```html
prompter> dump --where="block_id=32"
<div id="sbo-rt-content">
  <section
    data-type="colophon"
    epub:type="colophon"
    class="abouttheauthor"
    data-pdf-bookmark="About 
the Author"
  >
    <div class="colophon" id="idm45614675385984">
      <h1>About the Author</h1>
      <p>
        <strong>Zhamak Dehghani</strong> is a director of technology at
        Thoughtworks, focusing on distributed systems and data architecture in
        the enterprise. She’s a member of multiple technology advisory boards
        including Thoughtworks. Zhamak is an advocate for the decentralization
        of all things, including architecture, data, and ultimately power. She
        is the founder of data mesh.
      </p>
    </div>
  </section>
</div>
```

The `--where` clause can be any sqlite where clause for the columns 'block_id', 'block_tag', 'parent_block_id', 'group_id', 'group_tag', 'block', 'token_count'. When you look at the full output of the blocks command, you'll see 32 files, many of which (like a cover or index) are not things you want to send to an LLM. You could use the `--where` clause to just get the chapters:

```
blocks --where="block_tag like '%-ch%'"
```

## Creating new groups using transformations

The content downloaded from the platform contains a bunch of HTML markup that we don't want to send to the LLM. You can use the `transform` command to convert it to markdown:

```
transform --transformation=html2md
```

Rerun the blocks command just for the colophon:

```
prompter> blocks --where="block_tag like '%colo%'"
Current Blocks
┏━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━┓
┃ block_id ┃ block_tag ┃ parent_block_id ┃ group_id ┃ group_tag ┃ block ┃ token_count ┃
┡━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━┩
│ 63 │ 00030-colophon01-abou… │ 31 │ 2 │ me-wait-kind-gas │ # About the Author │ 55 │
│ │ │ │ │ │ **Zhamak Dehghani** │ │
│ 64 │ 00031-colophon02-colo… │ 32 │ 2 │ me-wait-kind-gas │ # Colophon The animal │ 175 │
│ │ │ │ │ │ on the cover of │ │
└──────────┴────────────────────────┴─────────────────┴──────────┴──────────────────┴─────────────────────────┴─────────────┘

2 blocks with 230 tokens.
Current group id = (2, 'me-wait-kind-gas')

The following fields available in --where clause:
['block_id', 'block_tag', 'parent_block_id', 'group_id', 'group_tag', 'block', 'token_count']

```

Here's the result of the conversion:

```

prompter> dump --where="block_tag like '%colophon01%'"

# About the Author

**Zhamak Dehghani** is a director of technology at Thoughtworks, focusing on distributed systems and data architecture in the
enterprise. She’s a member of multiple technology advisory boards including Thoughtworks. Zhamak is an advocate for the
decentralization of all things, including architecture, data, and ultimately power. She is the founder of data mesh.

```

A few things to note:

- the colophon now has a block_id of 63
- the current group id (shown at the end of the output of the blocks command) is now 2
- the group_tag has changed from `director-himself` to `me-wait-kind-gas`
- the HTML markup has been converted to markdown, which is a cleaner format to send to an LLM

The reason the group_id and block_id changed is that transformation in `prompter` do not update data -- it only adds new results. Each transformation creates a new group containing the corresponding blocks. The `groups` command will shows the groups different groups:

```
prompter> groups
                        Groups
┏━━━━┳━━━━━━━━━━━━━━━━━━┳━━━━━━━━━━━━━┳━━━━━━━━━━━━━━┓
┃ id ┃ group_tag        ┃ block_count ┃ prompt_count ┃
┡━━━━╇━━━━━━━━━━━━━━━━━━╇━━━━━━━━━━━━━╇━━━━━━━━━━━━━━┩
│ 1  │ director-himself │ 32          │ 0            │
│ 2  │ me-wait-kind-gas │ 32          │ 0            │
└────┴──────────────────┴─────────────┴──────────────┘

2 group(s) in total
Current group_id: (2, 'me-wait-kind-gas')
```

You can provide a `--group_tag` when doing transformations to name them yourself; otherwise, `prompter` will generate a random name for you.

As a last example of transformation, this command will split the current blocks into new blocks of 2000 characters with a group tag of `chunked`:

```
transform --transformation=token-split --group_tag=chunked
```
