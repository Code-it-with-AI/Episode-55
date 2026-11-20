# Episode-55: Agentic memory - to vectorize or not

📺 YouTube video: https://youtu.be/

🏠 Code it with AI Home Page: https://codeitwithai.com

---

I just got back from [Live 360 in Orlando](https://live360events.com/events/orlando-2026/home.aspx), and if I had to pick the one question I heard more than any other this week, it was some variation of "do I need a vector database?"

Sometimes it was "how do I give my agent memory?" or "what should I use for RAG?" or "we're thinking about standing up Pinecone (or Azure AI Search, or pgvector), is that the right call?" But it all comes back to the same basic question.

The answer, of course, is: sometimes.

I know, I know. Consultant answer. But it really does depend, and I think it is worth walking through _what_ it depends on, because the reasons are pretty concrete.

[STORY: a specific conversation or two from the conference that captures how people were asking this]

## What vectors are good at

Vectorization (text embedding, encoding, whatever term you prefer) takes a chunk of text and turns it into a long list of numbers that represents the _meaning_ of that text. Text with similar meaning ends up with similar numbers, even if the actual words are completely different.

That's super useful.

If a user asks "how do I keep someone from editing a record after it has been approved?" a vector search can find the document that talks about "authorization rules" and "read-only properties based on state", even though none of those words appear in the question. It finds _intent_. It finds _meaning_ across bodies of text.

Traditional keyword search can't do that. It would look for "editing", "record", and "approved" and probably come back with junk.

So yes, vectors are a real and valuable tool.

## What vectors are bad at

Now the flip side. Vector search is _terrible_ at finding discrete terms.

People's names. Specific medicines. Product numbers. Invoice numbers. Part numbers, SKUs, error codes, class names, ticket IDs. Anything where you need _that exact thing_, not something that kind of means the same thing.

Think about it. What is the "meaning" of invoice number 10443-B? To an embedding model it means "some invoice-ish identifier", and it means pretty much the same thing as 10443-C, or 20443-B. Those all land in roughly the same spot. Which is exactly what you _don't_ want when someone asks about a specific invoice.

Or medicines. Hydroxyzine and hydralazine are spelled similarly and are completely different drugs. "Close enough" is not good enough here. Not even a little.

Names are the same story. If I search for "Lhotka" I want documents that contain "Lhotka". I don't want documents about other people with vaguely similar names, or documents that are semantically "about" the same topics I write about.

This is where _lexical_ search wins. Plain old keyword matching, usually with a ranking algorithm like [BM25](https://en.wikipedia.org/wiki/Okapi_BM25) on top. It's been around forever, it is fast, it is cheap, and it finds exact terms very reliably. It just has no idea what anything _means_.

## Hybrid search

So the ideal, in most real-world cases, is to do both.

Run a lexical search and a semantic (vector) search, then combine the results. Documents that show up in both lists are very likely to be what you want. Documents that only show up in one list can still be useful, just ranked a bit lower.

This is what we do in the [CSLA MCP server](https://github.com/marimerllc/csla-mcp). When an agent (Copilot, Claude, whatever) calls the `search` tool, the server runs two searches in parallel:

1. A keyword search using BM25 over the CSLA example files, including 2-word phrase matching from adjacent words in the query.
1. A semantic search using embeddings (`text-embedding-3-large` via Azure OpenAI) against pre-generated embeddings for those same files.

Each set of scores is normalized to a 0 to 1 range, and then the two are averaged. A file with no score from one side counts as 0 for that side, so files that match _both_ ways float to the top. And if the embedding service isn't configured, the server just falls back to keyword-only search, which still works pretty well for a lot of CSLA questions because people tend to ask about specific types like `BusinessBase` or `DataPortal`.

> That normalization step is important. Raw cosine similarity scores from `text-embedding-3-large` tend to cluster around 0.3 to 0.5 for relevant documents, while normalized BM25 scores use the whole 0 to 1 range. Average those without normalizing and the keyword side wins basically every time.

A lot of the commercial products use a different approach to merge results, called Reciprocal Rank Fusion (RRF), which combines based on _rank position_ rather than raw score. Either way, the idea is the same.

## Or... just don't bother

All _that_ said, here's the part that I think surprised some folks at the conference.

Today's models often have a 1 million token context window. That is a _lot_ of text.

A page of markdown is maybe 500 words, give or take, which is something like 650 to 700 tokens. So 100 pages is roughly 70,000 tokens, and 200 pages is roughly 140,000 tokens. Even the more conservative 200K token context windows can hold that, and a 1M window can hold it many times over.

So if you have less than 100-200 pages of markdown (for example), you might just suck _all_ of it into the LLM context and not worry about it.

No vector database. No embedding pipeline. No chunking strategy. No re-indexing when a document changes. No arguing about chunk size or overlap. You just load the files and let the model read them.

And the model is _really_ good at reading. Better than any search algorithm, honestly, because it can do the semantic stuff _and_ the exact-term stuff at the same time, with full context around each piece of text instead of isolated chunks.

Is this free? No. Every call sends all those tokens, and that costs money and adds latency. But [prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching) (and similar features from other vendors) makes repeated use of the same big block of context a _lot_ cheaper. Compare that to the cost of standing up, maintaining, and keeping in sync a search infrastructure, and for small-to-medium bodies of knowledge the "just load it all" approach often wins.

I'll also say that models do get less precise as you pack more and more into the context window. Something buried deep in 800K tokens of text is more likely to get overlooked than something in a tight 20K token prompt. So "fits in the window" and "works well in the window" are not quite the same thing. My rough comfort zone is the 100-200 page range, but your mileage may vary, and you should test with your own content.

## What about agent memory?

This is where the question usually came from at the conference. People are building agents and want those agents to "remember" things across sessions.

A lot of agent memory is actually pretty small. User preferences. Facts about a project. Decisions that were made and why. Corrections ("don't do it that way, do it this way"). That kind of thing is often a few dozen or a few hundred short notes.

That fits in context _easily_.

Look at how Claude Code handles memory, for example. It is just markdown files on disk, plus an index file (`MEMORY.md`) that gets loaded into context at the start of each session. The agent reads the index, and if something looks relevant it reads the full file. No vectors at all.

And the coding agents in general (Claude Code, Copilot, Codex, etc.) mostly find things in your codebase using `grep` and file globbing. Lexical search! The agent does the "semantic" part itself by deciding _what_ to search for, looking at the results, and searching again if it didn't find what it wanted. The model is the semantic layer, and the tools are lexical.

That's a pretty powerful pattern, and it is worth considering before you reach for a vector database.

## So when _do_ you need one?

My general thinking goes something like this:

1. If your content is under a couple hundred pages, try loading it all into context first. Seriously. It might be all you need.
1. If your content is bigger than that, but the agent can search it with tools (grep, a SQL query, a full-text search index), try letting the agent do iterative lexical search. Again, the model brings the semantic understanding.
1. If your content is large _and_ users ask fuzzy, meaning-based questions where they don't know the right words, now you probably want vectors. And you almost certainly want hybrid search, not pure vector search.
1. If your content is full of names, part numbers, medicines, IDs, or other discrete terms, do _not_ use vector-only search. You'll be disappointed.

## Products that do vector and hybrid search

The good news is that you rarely need to build this from scratch. Most of the major data platforms now support vectors, and many support hybrid search directly. Here are some examples:

Search services:

- [Azure AI Search](https://learn.microsoft.com/azure/search/hybrid-search-overview) - full-text (BM25) and vector queries in a single request, merged with RRF ([ranking details](https://learn.microsoft.com/azure/search/hybrid-search-ranking)), plus an optional semantic re-ranker
- [Elasticsearch](https://www.elastic.co/docs/solutions/search/hybrid-semantic-text) - BM25 plus dense/sparse vectors, combined with [RRF](https://www.elastic.co/guide/en/elasticsearch/reference/current/rrf.html)
- [OpenSearch](https://docs.opensearch.org/latest/vector-search/ai-search/hybrid-search/index/) - the open source fork of Elasticsearch, with hybrid query support (also available as Amazon OpenSearch Service)
- [Google Vertex AI Vector Search](https://cloud.google.com/vertex-ai/docs/vector-search/about-hybrid-search) - combines dense and sparse (keyword-ish) embeddings
- [Amazon Bedrock Knowledge Bases](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html) - managed RAG on top of various vector stores, with a hybrid search option

Dedicated vector databases:

- [Pinecone](https://docs.pinecone.io/guides/search/hybrid-search) - managed vector database with hybrid (dense + sparse) search
- [Weaviate](https://docs.weaviate.io/weaviate/search/hybrid) - open source, with built-in hybrid search combining BM25 and vectors
- [Qdrant](https://qdrant.tech/documentation/concepts/hybrid-queries/) - open source, hybrid and multi-stage queries
- [Milvus](https://milvus.io/docs/multi-vector-search.md) - open source, with [full-text search](https://milvus.io/docs/full-text-search.md) and hybrid search
- [Chroma](https://www.trychroma.com/) - lightweight and popular for prototyping and local development

Databases you might _already_ have:

- [SQL Server 2025 and Azure SQL](https://learn.microsoft.com/sql/relational-databases/vectors/vectors-sql-server) - native vector data type and vector functions, alongside the full-text search that has been there for years
- [PostgreSQL with pgvector](https://github.com/pgvector/pgvector) - combine with Postgres' built-in [full-text search](https://www.postgresql.org/docs/current/textsearch.html) for hybrid
- [Azure Cosmos DB](https://learn.microsoft.com/azure/cosmos-db/gen-ai/hybrid-search) - vector search plus full-text search, with hybrid ranking
- [MongoDB Atlas](https://www.mongodb.com/docs/atlas/atlas-vector-search/hybrid-search/) - vector search and Atlas Search combined
- [Redis](https://redis.io/docs/latest/develop/ai/search-and-query/vectors/) - vector search alongside its full-text search capabilities
- [SQLite](https://www.sqlite.org/fts5.html) with FTS5 for full-text and [sqlite-vec](https://github.com/asg017/sqlite-vec) for vectors - great for a local agent where you don't want a server at all

> That last category is worth emphasizing. If you already run SQL Server, Postgres, Cosmos, or Mongo, you may not need a _new_ database at all. My career started before relational databases were popular, and I've seen plenty of "you need a whole new kind of database" waves come and go. Usually the existing databases absorb the good ideas. That seems to be happening again with vectors.

For .NET folks, [Microsoft.Extensions.VectorData](https://learn.microsoft.com/dotnet/ai/conceptual/vector-databases) gives you a common abstraction over many of these stores, so you aren't locked in to one vendor's SDK.

And if you want to read more about improving retrieval quality, Anthropic's [contextual retrieval](https://www.anthropic.com/news/contextual-retrieval) article is a good read. It shows how combining embeddings with BM25 (plus some other tricks) cuts retrieval failures pretty dramatically compared to embeddings alone.

## Summary

So, do you need a vector database?

Sometimes.

If you have a small-to-medium amount of content, try just putting it all in the context window. If you have a lot of content, try letting the agent search it with plain old lexical tools. If you need real semantic search over a large body of text, use vectors, but please use hybrid search so you don't lose the ability to find the exact names, numbers, and terms that your users actually care about.

And measure. Build a small set of real questions with known good answers, and test each approach against them. That will tell you more than any conference hallway conversation (including the ones with me :) ).

_This post was authored with the assistance of AI._
