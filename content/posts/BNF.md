+++
title = 'What is BNF Grammar and how to use it with LLMs'
date = 2024-02-02T00:17:08+01:00
draft = false
+++

# Introduction

Large Language Models (LLMs) have become 
cornerstones for a multitude of applications, ranging from automated content creation to sophisticated decision support systems.
As these models become increasingly integrated into complex software ecosystems, the necessity for structured, predictable, and
easily parseable output formats is more and more important. This is where the concept of using Backus–Naur Form (BNF)
Grammar comes into play, providing a powerful methodology for defining expected output formats from LLMs.

BNF Grammar, a notation technique for context-free grammars, has long been used in the field of computer science for the
specification of programming language syntax. It offers a concise and clear structure for defining the rules that govern the
formation of valid strings in a language. By applying BNF Grammar to the domain of LLMs, developers can precisely specify the
format of outputs, such as ensuring responses are structured in JSON or JSON arrays for example, thereby making the integration of LLMs
into downstream applications more robust.

This article aims to explore the intersection of BNF Grammar and Large Language Models, providing insights into how developers
can leverage BNF to define expected output formats from LLMs. We will cover the basics of BNF Grammar, its relevance
in the context of LLMs, and provide practical examples of how to use BNF to ensure that LLM responses meet specific
formatting requirements, such as JSON. By leveraging the power of BNF Grammar, developers can enhance the interoperability of
LLMs with existing data processing and analysis tools, unlocking new possibilities in
AI projects.

To read more about the BNF Grammar itself: [wikipedia](https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form)

# Using BNF Grammar as a hard constraint
LLama-cpp has a feature to integrate BNF Grammar as a hard constraint within Large Language Models (LLMs) marks a significant leap forward in the precision of
AI-generated content. This advanced technique fundamentally transforms the way LLMs generate responses by embedding BNF Grammar rules directly into
the next-token selection process. Instead of the model freely generating any plausible continuation based on its training, it is now constrained to select
strictly to the specified format, such as JSON structures, without requiring post-processing or manual correction.
only those tokens that align with the predefined grammar rules. This ensures that every piece of the output not only makes logical sense but also adheres
to the expected format.

By customizing the model's output generation in this manner, developers can effectively 'insert' custom code into the generative process, allowing for an
unprecedented level of control over the output. This capability is particularly groundbreaking for applications requiring stringent adherence to data formats
and structures, from automated code generation to dynamic content creation that fits into specific templates or schemas. The model, thus constrained, acts more
like a co-developer that understands not just the context of the query but also the exact structural requirements of the response, bridging the gap between
freeform generative AI and rule-based system outputs.

## With llama-cpp
Using LLama-cpp-python, it is easily possible to use a custom grammar:
```python
from llama_cpp.llama import Llama, LlamaGrammar

grammar_string = """
root ::= Character  
Character ::= '{' ws "name" ws ":" ws string ws ',' ws "age" ws ":" ws number ws ',' ws "armor" ws ":" ws Armor ws ',' ws "weapon" ws ":" ws Weapon ws '}'  
Armor ::= "Armor.leather" | "Armor.chainmail"  
Weapon ::= "Weapon.sword" | "Weapon.mace"  
string ::= '"' ([^"]*) '"'  
number ::= [0-9]+ ('.' [0-9]*)?  
datetime ::= string  
ws ::= [ \t\n]*  
boolean ::= 'true' | 'false'  
stringlist ::= '[' ws ']' | '[' ws string (ws ',' ws string)* ws ']'  
numberlist ::= '[' ws ']' | '[' ws number (ws ',' ws number)* ws ']'
"""
grammar = LlamaGrammar.from_string(grammar_string)

llm_model = Llama("llama-2-13b.Q8_0.gguf")

response = llm_model(
    "Generate one fantasy character",
    grammar=grammar, max_tokens=-1
)

```

Once parsed the LLM answer should be a valid JSON that looks like this:
```json
{
  "name": "Thalia Stormwind",
  "age": 27,
  "armor": "Armor.leather",
  "weapon": "Weapon.mace"
}
```

# Using BNF Grammar in Prompt
Integrating BNF Grammar with a hosted Large Language Model (LLM), such as those provided by OpenAI or other companies,
presents a unique set of challenges. As of the current state of technology, these platforms do not offer direct support
for passing custom grammar rules alongside API requests. This limitation means that developers seeking to enforce a specific
output format, like JSON, cannot directly embed BNF Grammar constraints into the model's generation process.
A potential workaround involves incorporating the grammar rules within the prompt itself, essentially instructing the model to
generate responses that adhere to these specifications. While this approach relies on the model's ability to
understand and apply the embedded grammar rules accurately, it offers a creative avenue for guiding the model towards
producing outputs in the desired format. However, the effectiveness of this method can vary, highlighting the need for
continued innovation in the field to support more sophisticated control over LLM outputs.

## Examples with GPT4
### BNF Grammar
The tests I ran with GPT4 yield interesting results but not perfect ones. 
For instance using this prompt returns an invalid json:
```
You will help generate fantasy characters for my game. Here is the BNF Grammar I am expecting for your answer:
root ::= Character  
Character ::= '{' ws "name" ws ":" ws string ws ',' ws "age" ws ":" ws number ws ',' ws "armor" ws ":" ws Armor ws ',' ws "weapon" ws ":" ws Weapon ws '}'  
Armor ::= "Armor.leather" | "Armor.chainmail"  
Weapon ::= "Weapon.sword" | "Weapon.mace"  
string ::= '"' ([^"]*) '"'  
number ::= [0-9]+ ('.' [0-9]*)?  
datetime ::= string  
ws ::= [ \t\n]*  
boolean ::= 'true' | 'false'  
stringlist ::= '[' ws ']' | '[' ws string (ws ',' ws string)* ws ']'  
numberlist ::= '[' ws ']' | '[' ws number (ws ',' ws number)* ws ']'
Generate one character with a cool name
```
The model just answered with the json without any filler text. This is nice but the json is invalid:
```json
{
  "name" : "Aldar Stormblade",
  "age" : 34,
  "armor" : Armor.chainmail,
  "weapon" : Weapon.sword
}

```
adding that the json should be valid solved the issue but it defeat the purpose of using the grammar in the first place:
```
You will help generate fantasy characters for my game. Here is the BNF Grammar I am expecting for your answer:
root ::= Character  
Character ::= '{' ws "name" ws ":" ws string ws ',' ws "age" ws ":" ws number ws ',' ws "armor" ws ":" ws Armor ws ',' ws "weapon" ws ":" ws Weapon ws '}'  
Armor ::= "Armor.leather" | "Armor.chainmail"  
Weapon ::= "Weapon.sword" | "Weapon.mace"  
string ::= '"' ([^"]*) '"'  
number ::= [0-9]+ ('.' [0-9]*)?  
datetime ::= string  
ws ::= [ \t\n]*  
boolean ::= 'true' | 'false'  
stringlist ::= '[' ws ']' | '[' ws string (ws ',' ws string)* ws ']'  
numberlist ::= '[' ws ']' | '[' ws number (ws ',' ws number)* ws ']'
Your answer should be valid json.
Generate one character with a cool name
```
### JSON Schema
An other way of specifying that a JSON is expected is by using a JSON Schema. Of course it is less flexible than BNF as only JSON can be used.
So let's make a last test with JSON Schema instead of BNF:
```
You will help generate fantasy characters for my game. 
I expect your answer to follow this json schema:
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "CharacterList",
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "name": {
        "type": "string"
      },
      "age": {
        "type": "number"
      },
      "armor": {
        "type": "string",
        "enum": ["Armor.leather", "Armor.chainmail"]
      },
      "weapon": {
        "type": "string",
        "enum": ["Weapon.sword", "Weapon.mace"]
      }
    },
    "required": ["name", "age", "armor", "weapon"]
  }
}
Generate four fantasy characters
```
and the result looks perfect:
```json
[
  {
    "name": "Eldrin the Brave",
    "age": 34,
    "armor": "Armor.leather",
    "weapon": "Weapon.sword"
  },
  {
    "name": "Mira of the Forest",
    "age": 26,
    "armor": "Armor.leather",
    "weapon": "Weapon.mace"
  },
  {
    "name": "Thorn Ironhand",
    "age": 40,
    "armor": "Armor.chainmail",
    "weapon": "Weapon.mace"
  },
  {
    "name": "Lilith Shadowalker",
    "age": 29,
    "armor": "Armor.chainmail",
    "weapon": "Weapon.sword"
  }
]
```
## Examples with Mixtral 8x7b on Replicate.com
### BNF Grammar
With Mixtral, results are not as good. 
Using the first prompt yields:
```
Sure! Here is a character generated using your BNF grammar:

Character = {
name : "Thrain Ironjaw",
age : 45,
armor : Armor.chainmail,
weapon : Weapon.mace
}

This character's name is Thrain Ironjaw, they are 45 years old, and they are wearing chainmail armor and wielding a mace.
```
Not only the json is not valid but there is filler text before and after the json. Let's change the prompt:
```
You will help generate fantasy characters for my game. Here is the BNF Grammar I am expecting for your answer, please do not add any text to your answer. I just want the following grammar:
root ::= Character  
Character ::= '{' ws "name" ws ":" ws string ws ',' ws "age" ws ":" ws number ws ',' ws "armor" ws ":" ws Armor ws ',' ws "weapon" ws ":" ws Weapon ws '}'  
Armor ::= "Armor.leather" | "Armor.chainmail"  
Weapon ::= "Weapon.sword" | "Weapon.mace"  
string ::= '"' ([^"]*) '"'  
number ::= [0-9]+ ('.' [0-9]*)?  
datetime ::= string  
ws ::= [ \t\n]*  
boolean ::= 'true' | 'false'  
stringlist ::= '[' ws ']' | '[' ws string (ws ',' ws string)* ws ']'  
numberlist ::= '[' ws ']' | '[' ws number (ws ',' ws number)* ws ']'
Generate one character with a fantasy style name
```

and the result looks like valid json to me:
```json
{
"name": "Thrain Ironshield",
"age": 200,
"armor": "Armor.chainmail",
"weapon": "Weapon.mace"
}

```

Let's see if it is possible to genereate many characters in a json array:

### JSON Schema
And with JSON Schema

```
You will help generate fantasy characters for my game. 
I expect your answer to follow this json schema:
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "CharacterList",
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "name": {
        "type": "string"
      },
      "age": {
        "type": "number"
      },
      "armor": {
        "type": "string",
        "enum": ["Armor.leather", "Armor.chainmail"]
      },
      "weapon": {
        "type": "string",
        "enum": ["Weapon.sword", "Weapon.mace"]
      }
    },
    "required": ["name", "age", "armor", "weapon"]
  }
}
Generate four fantasy characters
```
The result is not strictly valid. However it is composed of valid (hence parsable) JSONs:

```
1. {
"name": "Eldrith Ironheart",
"age": 34,
"armor": "Armor.chainmail",
"weapon": "Weapon.sword"
}

2. {
"name": "Thora Silvermoon",
"age": 27,
"armor": "Armor.leather",
"weapon": "Weapon.mace"
}

3. {
"name": "Grimbeard Oakmane",
"age": 42,
"armor": "Armor.chainmail",
"weapon": "Weapon.sword"
}

4. {
"name": "Luna Whisperwind",
"age": 29,
"armor": "Armor.leather",
"weapon": "Weapon.mace"
}
```

# Conclusion

Concluding our exploration into imposing structured formats on Large Language Model (LLM) outputs,
it's evident that the most effective and reliable method hinges on the integration of BNF Grammar within the LLama-cpp framework.
This approach not only allows for precise control over the output by embedding specific grammar constraints directly into the model's token selection process
but also ensures consistency and adherence to desired formats, such as JSON. 

Alternatively, while it is feasible to guide models towards generating outputs in custom formats by specifying these requirements within the prompt, this
method lacks the robustness and reliability necessary for production environments. Although GPT-4 and similar models can indeed generate outputs that
approximate the desired structure, the inherent variability and potential for deviation make this approach less preferable for applications requiring strict
format compliance.

In summary, for developers and organizations looking to leverage LLMs for tasks where output structure is paramount, the combination of BNF Grammar
with LLama-cpp stands out as the superior choice. It offers a blend of precision, reliability, and control unmatched by prompt-based formatting requests,
underscoring the importance of continued innovation in tools that enable more granular control over AI-generated content.
