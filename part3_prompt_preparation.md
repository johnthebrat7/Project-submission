# Prompt Preparation Document — MetaGPT PR#1450

# 3.1.1 Repository Context


MetaGPT is an open source multi agent LLM platform managed by the FoundationAgents group. The platform replicates an entire software development team with every member being an LLM agent responsible for executing a Standard Operating Procedure: Product Manager, Architect, Project Manager, Engineer, and QA Engineer. The philosophical principle underpinning the platform, namely Code = SOP(Team), is the notion that materializing software companies' SOPs via LLM agents can result in a cohesive output delivered by a team from a natural language requirement. With one line such as "Build a 2048 game", MetaGPT can generate user stories, competitor analysis, requirement document, system design, data structure definition, API specifications, source code, and testing.

Target users belong to one of three categories: (1) AI/ML researchers investigating multi-agent cooperation, specialization, and the LLM-as-Orchestrator paradigm; (2) developers interested in quickly prototyping their ideas by describing them in natural language and employing the resulting code repository as a starting point; and (3) the broader open-source community exploring autonomous agents' workflows. The primary publication was accepted at ICLR 2024, while a companion paper (AFlow) received an oral presentation slot at ICLR 2025.

Problem space : autonomous software development and natural language programming. There are three related sub-problems: the problem of parsing the user’s fuzzy goal into concrete deliverables, the problem of coordinating different agents operating on the same artifact (PRD -> design -> code -> tests), and the problem of abstracting over LLM providers such that the agent code works on OpenAI, Azure, Anthropic, Google Gemini, Ollama-based local models, AWS Bedrock, and other providers. The provider abstraction in metagpt/provider/, precisely where this PR is relevant, provides a common interface for all backends, hiding their idiosyncrasies, such as authentication, prompt format, token limits, and streaming.



# 3.1.2 Pull Request Description



PR#1450, feat(bedrock): Temporary AWS credentials via env vars + supported models update, modernises MetaGPT's Amazon Bedrock provider on two fronts.
Specific changes. 

(a) The Bedrock provider is updated so AWS credentials can be supplied with the help of the standard AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SESSION_TOKEN, and AWS_DEFAULT_REGION environment variables .Not only with the help of  fields in ~/.metagpt/config2.yaml.

(b) The supported models catalogue in metagpt/provider/bedrock/utils.py is updated: AI21 Jamba Instruct, Amazon Titan Text Premier, Cohere Command R and Command R+, and Mistral Large 2 models are added with the right max_tokens; EOL models AI21 J2 Mid and J2 Ultra (us west 2), and the LLaMA 2 models from Meta are deleted. New prompt formatting utilities are added for the new models, and there is a fix in the constant NOT_SUUPORT_STREAM_MODELS.


** Why there is a need for these changes ? **According to the security guidelines of AWS itself, temporary credentials generated through STS must be used in place of static keys. Static keys stored within a YAML file located in a user’s home directory are clumsy, incompatible with SSO/IAM role/CI systems, and strongly discouraged by AWS itself. In the code before PR submission, AWS_SESSION_TOKEN wasn’t honored, making these systems unusable. Moreover, the model catalog was outdated, with some new models missing and others being EOLs listed.

Old versus new behavior. Users were required to provide aws_access_key_id and aws_secret_access_key manually in the YAML configuration file for MetaGPT. Furthermore, only persistent IAM user credentials would be accepted. However, MetaGPT will try boto3’s default credential resolution flow if YAML fields for accessing AWS are missing – reading environment variables, the ~/.aws/credentials file, or instance credentials in the case of EC2 and ECS, and single sign on sessions.


# 3.1.3 Acceptance Criteria




1. When the user sets AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, and AWS_DEFAULT_REGION as environment variables and leaves aws_access_key_id/aws_secret_access_key absent from config2.yaml, the Bedrock provider should authenticate successfully and respond to a hello-world prompt.

2. When the user additionally sets AWS_SESSION_TOKEN , the provider should forward the session token to boto3 transparently and succeed against Bedrock with no further configuration.

3. When the user does specify aws_access_key_id and aws_secret_access_key in config2.yaml , the provider should continue to authenticate exactly as it did before this PR backward compatibility is preserved with zero behavioural drift.

4. All existing tests under tests/metagpt/provider/ continue to pass; new unit tests cover both the env var credential path and the new model entries, using mocked boto3 clients so tests do not make real AWS calls.

5. In case the user sets the model to “cohere.command-r-plus-v1:0” in the YAML file and executes the generation call, the prompt formatter utility is expected to serialise the chat in the required Command R+ format and provide a valid response from the provider side.



# 3.1.4 Edge Cases



1. Expired temporary credentials. The user provides STS credentials through environment variables that expire during the session. If boto3 throws ExpiredTokenException, then MetaGPT needs to return an understandable error message that is not buried within another LLM exception but instead simply states “Expired AWS credentials, please renew the STS session.”

2. Partial credential configuration. User sets AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY in the environment but forgets AWS_DEFAULT_REGION, and there is no region in config2.yaml either.The provider needs to fail with a specific error that can be acted upon referring to both AWS_DEFAULT_REGION and the YAML parameter that would solve the problem instead of failing silently within boto3.

3. The EOL Model is still configured. The user who has upgraded from an older version of MetaGPT still has the model configuration as "meta.llama2-70b-chat-v1" in the YAML file. The service provider must give out a deprecation/removal warning indicating the equivalent model in Llama 3 instead.



# 3.1.5 Initial Prompt



You are an experienced Python contributor to the MetaGPT open-source project (https://github.com/FoundationAgents/MetaGPT). MetaGPT is a multi agent LLM framework where the metagpt/provider/ package implements pluggable LLM backends OpenAI, Azure, Anthropic, Google, Ollama, AWS Bedrock, and others all conforming to a common interface.
Your task is to implement PR#1450: extend the Bedrock provider to support AWS credentials supplied via the standard AWS environment variables (including temporary STS credentials), and refresh the supported-models catalogue.

Scope of work

In metagpt/provider/bedrock/bedrock_provider.py, modify the boto3 client initialisation so that:

If aws_access_key_id and aws_secret_access_key are present in MetaGPT's LLMConfig, use them (preserve current behaviour).
If they are absent, do not raise let boto3's default credential resolution chain pick them up from the environment (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SESSION_TOKEN, AWS_DEFAULT_REGION), ~/.aws/credentials, IAM roles, or SSO.
The region should likewise fall back to AWS_DEFAULT_REGION when not given in config.


In metagpt/provider/bedrock/utils.py, update the supported models dictionary:

Add: ai21.jamba-instruct-v1:0, amazon.titan-text-premier-v1:0, cohere.command-r-v1:0, cohere.command-r-plus-v1:0, mistral.mistral-large-2407-v1:0, each with correct max_tokens per AWS documentation.
Remove or comment out: AI21 J2 Mid/Ultra (us-west-2) and the Meta Llama 2 family these are EOL.
Where new families need a custom prompt format (Cohere Command R is chat style; Jamba uses a messages array), extend the existing message_to_prompt helpers.
Fix the typo NOT_SUUPORT_STREAM_MODELS → NOT_SUPPORT_STREAM_MODELS everywhere it is referenced.



Acceptance criteria (all must hold)

Setting only AWS_ACCESS_KEY_ID + AWS_SECRET_ACCESS_KEY + AWS_DEFAULT_REGION as environment variables (no YAML credentials) lets a hello-world Bedrock call succeed.
Adding AWS_SESSION_TOKEN to that environment also succeeds (temporary credentials work).
Users who keep credentials in config2.yaml see zero behavioural change.
Calls to each newly added model succeed with prompts formatted for that family.
All previously passing tests under tests/metagpt/provider/ still pass; new tests cover the env-var path and the new model entries.

Edge cases to handle

Missing region with no environment variable fallback → raise a clear error naming AWS_DEFAULT_REGION.
Expired STS token → propagate boto3's ExpiredTokenException with a user readable message; do not retry indefinitely.
Unknown model ID → fail fast with a listing of supported IDs.
EOL model ID still in a user's YAML → emit a deprecation warning suggesting a replacement.

Testing requirements

Use pytest. Mock boto3.client with unittest.mock so tests make no real AWS calls.
Add at least one test per new model entry verifying max_tokens lookup and prompt formatting.
Add tests that monkeypatch os.environ to simulate environment variable only credentials, plus a test verifying YAML credentials take precedence when both sources are present.

Follow MetaGPT's existing style (see ruff.toml), keep type hints consistent with the rest of metagpt/provider/, and update docstrings. Reference the OpenAI and Anthropic providers as patterns for clean implementation.
