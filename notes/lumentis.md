---
title: How Lumentis works
tags:
  - llms
  - ai
emoji: 👋
link: https://github.com/josh-cooper/lumentis-kindle
modified: 2024-06-07
---
# Outline
## Workflow:
Generate an outline, and until the user is happy:
  1. Eval the outline.
     - At the moment this is just to validate the JSON structure.
     2. There is an attempt to do partial parsing on an incomplete JSON, and prompt the LLM to continue.
  2. Ask the user to manually set which sections they want to keep, add or change.
  3. Ask the user for any other requested changes (in natural language)
  4. Re-generate the outline

Note that the outline is generated with metadata, inlcuding a title, permalink, short description, a list of "key things to cover", a list of subsections. The subsections are an array of strings and don't contain metadata.

## Generating the outline

Compile a promptfrom the following:
  - Title: Short user generated title/name, or can be auto-generated. E.g. "Balanced Information Diet Principle".
  - Primary Source
  - Description: Short user provided description, or can be auto-generated. Example below.
  - Core Themes: Short user provided themes or keywords, or can be auto-generated. Example below.
  - Itended Audience: Short user provided list, or can be auto-generated. Example below.
  - Ambiguities: Use GPT to generate a list of questions to ask the user to clarify the outline.
  - Writing Example: User provided example, but no option to auto-generate.

## Examples and Prompts

### Description

#### Prompt

```
<PrimarySource>
{full_source}
</PrimarySource>

Please provide a three sentence description of the information in PrimarySource. Is this a conversation transcript, an article, etc? What is it about? what are the key themes and who is this likely by and for? No newlines.
```

#### Example
**PrimarySource** is a transcript of a monologue, likely from a podcast or a lecture, discussing how our current model of information consumption is flawed and proposes a philosophy called "intentional information" to improve life quality by being selective about the information we consume and the mediums we use. Key themes include the impact of technology on information perception, and the historical and neurological aspects of how we process information. It appears to be by an expert in productivity or technology, aimed at an audience interested in mindfulness, self-improvement, and better information management strategies.

### Title

#### Prompt

```
<PrimarySource>
{full_source}
</PrimarySource>

{description}

Please generate up to 10 possible names for documentation we want to build, for the data in PrimarySource. Return them as a JSON array of strings without markdown code blocks.
```

### Core Themes

#### Prompt
```
<PrimarySource>
{full_source}
</PrimarySource>

{description}

Please generate up to 10 possible keywords referring to industries, technologies, people or other themes for the data in PrimarySource. Return them as a JSON array of strings without markdown code blocks.
```

#### Example

 - intentional information
 - technology criticism
 - information technology
 - epistemic environment
 - neuroscience
 - social media
 - media impact

### Audience

#### Prompt
```
<PrimarySource>
{full_source}
</PrimarySource>

{description}

Please generate up to 10 words describing the intended audience for creating documentation from the data in PrimarySource (which level, what type of job, etc). Return them as a JSON array of strings without markdown code blocks.
```

#### Example
 - knowledge workers
 - executives
 - students
 - tech enthusiasts
 - self-improvement advocates
 - productivity experts
 - academics


### Ambiguity

#### Prompt

```
<PrimarySource>
{full_source}
</PrimarySource>

{description}

We want to build proper comprehensive docs for what's in PrimarySource. Can you give me a JSON array of strings, of 10 questions about things that might be confusing, need more explanation, or color?
```

Or to iterate off an initial set:

```
<PrimarySource>
{full_source}
</PrimarySource>

{description}

Here are some questions already asked and partially answered.
<PastQuestions>
{past_questions}
</PastQuestions>

We want to build proper comprehensive docs for what's in PrimarySource. Can you give me a JSON array of strings, of 10 questions about things that might be confusing, need more explanation, or color?
```

Example:
<details>
  <summary>Show</summary>

Here are some questions:

1. What are the main flaws in our current model of information consumption that the speaker highlights?

Answer:


2. Can you explain the concept of 'intentional information' in more detail?

Answer:


3. How does the medium through which we receive information shape our understanding of that information?

Answer:


4. What historical examples does the speaker provide to illustrate the impact of technology on information perception?

Answer:


5. What role does the concept of 'epistemic environment' play in the speaker's argument?

Answer:


6. Could you provide more information on the book 'The Great Partnership' by Jonathan Sacks and its relevance to the topic?

Answer:

...

</details>

### Outline: Bringing it altogether

#### Initial Prompt
Just dump it all in:

```
<PrimarySource>
{full_source}
</PrimarySource>

{description} # note there's still no delimiting of this

Here are some questions that can help with any ambiguity or things that need explaining in PrimarySource. Feel free to use your knowledge for the things I haven't answered.
<Questions>
{ambiguity_qa}
</Questions>

Here are some of the themes covered in PrimarySource: # in addition to the "key themes" already asked for in the description?
<Themes>
{themes}
</Themes>

Here are the intended audience for the documentation:
<Audience>
{audience}
</Audience>

Here's an example of the kind of writing we're looking for:
<WritingExample>
{writing_example}
</WritingExample>

Let's use PrimarySource to generate good documentation. Can you generate a JSON of the outline of this documentation (sections, subsections, permalinks, etc) following this typespec? Ideally the first section doens't have any subsections.

\`\`\`typescript
type OutlineSection = {
  title: string;
  permalink: string; // something easier to use as identifier
  singleSentenceDescription: string;
  keythingsToCover:string[]; very short strings to list things you want to make sure are covered in this section.
  subsections?: OutlineSection[];
};

type Outline = {
  title: string;
  sections: OutlineSection[];
};
```

#### Regenerating
1. Re-add the entire initial prompt as an initial user message.
2. Add the *user updated* outline as an assistant message.
3. Add a user message with whatever the user requested to change:
```
Can you regenerate the outline with the following requests or new sections? 
{requested_changes}
Follow the Outline typespec.
```

Note that the outline typespec can only be sourced from the initial prompt. This might only work because its placed right at the end of that prompt and the section data is not super long either.

### Misc

#### System Prompt
There's an option to set this in the LLM call functions, but it does not appear to be used.

#### Array Output

##### System prompt
Output as JSON with the following system prompt.

```
{existing system prompt}

Please wrap the returned array in a JSON object with a key of 'results' to ensure proper parsing. Eg: structure the returned object as `{ \"results\": [ ... ]}
```

#### JSON Output

##### OpenAI
Set `response_format` to "json".

##### Anthropic
Set an initial assistant prompt of `"{"` or `"["`.

# Document
Once we have an outline each section can be indvidiually generated.

## Workflow
1. Recursively generate a page writing prompt for each page (i.e. section or sub-section)
2. Call each page's prompt individually
3. Eval? Reflection? Doesn't look like it

## Examples and Prompts
### Page Writing
1. Add an initial user message with the full outline generation prompt
2. Add an assistant prompt with the actual outline
3. Add a user prompt describing how to generate the page:

```
Now we're going to specifically write the section {title} (permalink: {permalink}) in mdx, following these guidelines:

1. Write in mdx, with appropriate formatting (bold, italics, headings, bullet points, <Callout>, <Steps> etc). We're going to use this as a page in nextra-docs. Use Callouts when needed. Steps look like this:
<Steps>
### Step 1

Contents

### Step 2

Contents
</Steps>
2. Add mermaid diagrams in markdown (```mermaid) and latex (surrounded by $) when needed.
3. Write only the section, no need to talk to me when you're writing it.
4. Write it as an expert in the themes, but for the intended audience
5. Don't put mdx code blocks around the output, just start writing.
6. Each subsection and section will have its own page. Just write the specific one you're asked to write.
7. Be casually direct, confident and straightforward. Use appropriate examples when needed.
8. Add links to subsections or other sections. The links should be in the format of [linktext](/section-permalink/subsection-permalink). Use / as the permalink for the intro section.
9. Provide examples when needed. Use the source when you can, quoted (if you can attribute) or otherwise.
10. Make sure to start headings in each section and subsection at the top level (#).
11. The subsections {comma_separated_subsections} will be written later, and don't need to be elaborated here.
```

Some interesting things to note:
 - The Steps formatting seems like something I would think is a lot of unnecessary complexity in the prompt. Maybe this helps infer other, non-standard markdown formatting. More likely we need to parse this later when formatting for web.
 - Inclusion of Guideline (2) is optional
 - The inclusion of Guideline (11) is only included if there are subsectiosn present
 - This, like many of the other prompts, seems carefully constructed. But I wonder how much was decided at the outset, versus updated experimentally as failure modes were uncovered.
