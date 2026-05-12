** Since MetaGPT has highest involvement of python I will be evaluating it for this particular section. **

PR 1:#1224 — Fixing the potential duplicate embeddings in the RAG module

# SUMMARY :

A minor inefficiency in MetaGPT's RAG module is that in cases where users pass in retriever_configs for SimpleEngine.from_docs(), then the index inside get_retriever() is rebuilt, as opposed to using the already-computed embedding vectors. The effect is that there is a redundancy in the form of re-embedding of documents, leading to unnecessary spending on embedding API calls (that incur costs and time), as well as extra vector storage. This PR removes the redundancy and, as an additional benefit, refactors SimpleEngine such that it uses a pipeline-based approach (as opposed to precomputed VectorStoreIndex). In addition, it enhances the base factory error handling, adds support to the retriever factory for fetching or building indexes depending on configuration, and fixes affected unit tests.

---------------------------------------------------


# TECHINCAL CHANGES


1. metagpt/rag/engines/simple.py — removes direct VectorStoreIndex dependency, introduces a transformations-based architecture, adds a new _from_nodes constructor path.

2. metagpt/rag/factories/retriever.py — adds decorators that drive a dynamic build-vs-retrieve decision based on config

3. metagpt/rag/factories/base.py — improves ConfigBasedFactory error messages , _val_from_config_or_kwargs now returns None instead of raising KeyError

4. examples/rag_pipeline.py — adds docstrings to RAGExample and wraps async methods in exception-handling decorators

5. tests/metagpt/rag/engines/test_simple.py — replaces VectorStoreIndex coupled mocks with transformations path mocks

6. tests/metagpt/rag/factories/test_base.py — updated to assert None return rather than expected exception.

7. tests/metagpt/rag/factories/test_retriever.py — new tests covering the dynamic build-or-retrieve branches.


-----------------------------------------------------
# IMPLEMENTATION APPROACH

The original SimpleEngine.from_docs() always materialised a VectorStoreIndex upfront, then get_retriever() re derived an index from the supplied retriever_configs which silently re embedded the documents. The refactor breaks that loop by treating ingestion as a sequence of transformations (node parsers, embedding generators, etc.) rather than a finished index. The engine now keeps the nodes/transformations as the canonical artifact and constructs a VectorStoreIndex lazily and only once via the new _from_nodes path. In the retriever factory, decorators inspect the supplied config and dispatch to one of two branches: (a) if an index already exists for these embeddings, fetch it; (b) only build a new index when a different vector store or transformation set is requested. The base factory complements this by switching _val_from_config_or_kwargs from raising KeyError to returning None, which lets the decorator logic use clean conditional checks (if value is None: ..) rather than try/except scaffolding. The tests mirror the structural shift: old mocks tied to VectorStoreIndex are deleted, and new tests directly exercise the build vs fetch branches of the decorator to lock in the no-duplication invariant.

-------------------------------------------------

# POTENTIAL IMPACT

The blast radius is the RAG subsystem: metagpt/rag/engines/simple.py and the factory layer under metagpt/rag/factories/. Anyone instantiating SimpleEngine.from_docs() benefits from fewer embedding API calls and lower first query latency. Risk: callers who relied on VectorStoreIndex being directly accessible on the engine instance, or on _val_from_config_or_kwargs raising on missing keys, will observe behavioural changes. Roles that depend on RAG (e.g. Researcher, Engineer with context retrieval) and the reference examples/rag_pipeline.py are the main downstream consumers to re-verify.

-----------------------------------------------------------

PR 2:#1450 — feat(bedrock): Temporary AWS credentials via environment variables + supported models update

# SUMMARY:

MetaGPT's Amazon Bedrock provider previously required users to hard code AWS access and secret keys inside ~/.metagpt/config2.yaml. That was awkward for two reasons: it ignored AWS's standard environment variable convention (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SESSION_TOKEN, AWS_DEFAULT_REGION) that the rest of the AWS ecosystem assumes, and it offered no path to use temporary credentials issued by AWS STS the recommended way to operate against cloud accounts, especially in production or SSO managed environments. This PR fixes both: the Bedrock provider can now bootstrap entirely from the standard AWS environment variables, including session tokens, with no MetaGPT specific keys required in YAML. It also updates the model catalogue adding newer Bedrock hosted models (Jamba-Instruct, Titan Text Premier, Cohere Command R/R+, Mistral Large 2) and pruning legacy/EOL entries (AI21 J2 Mid/Ultra, Meta Llama 2).

-------------------------------------------

# TECHINCAL CHANGES

1. metagpt/provider/bedrock/bedrock_provider.py — relaxes credential requirements so that aws_access_key_id / aws_secret_access_key become optional in YAML; lets boto3 pick up AWS_* environment variables and AWS_SESSION_TOKEN for temporary credentials

2. metagpt/provider/bedrock/utils.py — updates the supported-models map: adds new model IDs with their correct max_tokens; comments out / removes EOL entries; corrects previously-wrong token limits; fixes a typo in the constant name NOT_SUUPORT_STREAM_MODELS → NOT_SUPPORT_STREAM_MODELS

3. Provider-level message-to-prompt formatting tweaks to accommodate the new Command R/R+ and Jamba model families

4. README / docstring example block (in the PR description) showing a credential-less llm: YAML configuration paired with the four AWS_* env vars

-----------------------------------------------------

# IMPLEMENTATION APPROACH

The change cleanly separates into a credentials path and a model catalogue path. For credentials, the provider previously instantiated a boto3 client by explicitly passing access key, secret key, and region read from MetaGPT's LLMConfig. The refactor inverts the default: if those YAML fields are present they are still honoured (backward compatibility), but when absent, the provider falls through to boto3's native credential resolution chain which automatically picks up AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SESSION_TOKEN, and AWS_DEFAULT_REGION from the environment, or any other source boto3 supports (IAM roles, aws/credentials, SSO). That single change unlocks temporary STS credentials for free, because boto3 already knows how to use a session token whenever one is present no further code is needed. On the model side, the constants dictionary in utils.py mapping each Bedrock model ID to its provider specific quirks (max tokens, streaming support, prompt format function) is updated entry by entry: new Anthropic/Cohere/Mistral/Amazon/AI21 model IDs are added with their correct max_tokens, and EOL entries are commented out so they no longer appear in user facing model lists. Reviewer feedback in the PR thread also led to pruning smaller (<30B) Llama 3 variants to nudge users toward stronger models.

---------------------------------------------

# POTENTIAL IMPACT 

The effect is confined to metagpt/provider/bedrock/. Users of other providers (OpenAI, Azure, Anthropic-direct, Ollama, Gemini, etc.) are untouched. Existing Bedrock users with explicit YAML credentials keep working (backward-compatible). New upside: MetaGPT is now usable from EC2/ECS with IAM roles, from AWS SSO sessions, and from CI environments without secrets-in-files; the catalogue update unlocks newer model families like Mistral Large 2 and Cohere Command R+. Risk: anyone still pointing at a removed legacy model ID (J2, Llama 2) will need to switch.


