::

    Status: Draft
    Type: Feature
    Created: 2024-12-13
    Authors: Andrey Buzin <andrey@edgedb.com>


=========================
RFC 1028: Extended AI API
=========================

Abstract
========

This RFC describes the proposed updates to the Gel server and the Python
client library that would provide convenience functions for various
AI-related workflows.

Motivation
==========

The existing Python API for AI functionality presents a steep learning
curve for =ew users looking to build their LLM-enabled applications.

It has a peculiar interface and requires a lot of practical experience
with Gel in order to build something that isn't a part of the
Getting Started tutorial. It also doesn't expose any of the intermediate
steps for creating a RAG application, thus making it difficult to
customize them. Finally, in makes it difficult to gauge Gel's AI
capabilities for somebody assessing it from the outside with no
prior knowledge.

To this end, here are the goals of the redesigned interface:

- Match user expectations based on using similar products.
- Make behavior more transparent by shifting configuration to parameters
  as opposed to a context object and method chaining.
- Expose similarity search and LLM calling separately to simplify
  building custom RAG workflows.

Also a part of this proposal is the GelVectorstore class that represents
ext::vectorstore within the AI package.

This extension was meant to be used with third-party framework
integrations. However, it's can be valuable on it's own merit by
providing pre-built tools to create vector search applications such as
image search. Having a first-party binding would also enable users to
use the extension in a framework-agnostic way.


Implementation
==============

Extension AI Binding
--------------------

The proposed API splits search functionality from the RAG so it can be
used on its own without having to carry around the LLM configuration.

Apart from the vector search method that uses `ext::ai` functionality,
it also provides ways to search using `fts` and `pg_trgm`. While not
directly tied to AI, those could be (and commonly are) used alongside
vector search in RAG applications. The API also has a hybrid search
method, which would implement a combination of vector and full-text
search.

All of the search methods can be called with the same set of parameters,
all of which make up the EdgeQL query used to retrieve the results.
Supplying query configuration in this way makes it easier to control
the search behavior. It also offers a way to do it that may be easier
for a new user to digest than a raw EdgeQL query.

The last set of methods provides a way to quickly generate an embedding
for a string using whatever model is being used in the database at the
moment. Nothing incredibly advanced here, it's just handy when building
an application,


.. code-block:: python
    class GelAI:
        """
        The primary interface for Gel's ext::ai functionality.

        Enables common retrieval workflows, provides methods to query instance cofiguration
        for embedding models.
        """

        def __init__(self):
            pass

        def semantic_search(
            self,
            text_query: str,
            edgeql_query: str,
            shape: str,
            filter: str,
            group: str,
            limit: int,
            offset: int,
        ) -> List[Dict[str, Any]]:
            """
            Perform vector similarity search for a string

            Args:
                text_query (str):   The string to embed and search for.
                edgeql_query (str): The EdgeQL query that resolves into a type.
                                    The type must have ext::ai::index on it.
                shape (str):        The shape to apply to objects before returning them.
                                    By default contains a single splat, the distance and the embedding.
                filter (str):       The filter statement to apply to objects before ranking them by distance.
                group (str):        The group statement to apply to objects before returning them.
                limit (int):        The maximum number of objects to return.
                offset (int):       The offset to apply to the set.

            Return:
                A list of json objects contating whatever was found by similarity search,
                along with distances and respective embeddings.

            Example:
                >>> gel_ai.semantic_search(
                        text_query="jupyter",
                        edgeql_query="Astronomy",
                        group=".solar_system",
                        limit=5,
                    )

                [
                    { whatever the object shape is, "distance": 0.12345, "embedding": [ ... ]}
                ]
            """
            pass

        def fulltext_search(self):
            """
            Perform full-text search for a string using Gel's built in FTS functionality.

            Requires the FTS index to have been specified in the schema.
            """
            pass

        def trigram_search(self):
            """
            Perform trigram search for a string using Gel's pg_trgm integration.
            """
            pass

        def hybrid_search(self):
            """
            Perform semantic search and full-text search for the same string and fuse the results.

            Requires both indices to have been specified in the schema for the selected type.
            """
            pass

        def search_by_vector(
            self,
            vector: List[float],
            edgeql_query: str,
            shape: str,
            filter: str,
            group: str,
            limit: int,
            offset: int,
        ) -> List[Dict[str, Any]]:
            """
            Perform vector similarity search for a vector provided by the user.
            """
            pass

        def generate_embedding(
            self,
            text_query: str,
            model: str,
        ):
            """
            Use Gel's server as a passthough to generate an embedding for a text string.

            Requires the model whos name has been passed as the argument to have been
            configured in project's instance.
            """
            pass

        def generate_embedding_like(
            self,
            text_query: str,
            edgeql_query: str,
        ):
            """
            Use Gel's server as a passthough to generate an embedding for a text string.

            Do it using whatever model has been used to index the type specified by the provided query.
            """
            pass

        @property
        def embedding_model(self, edgeql_query: str) -> Dict[str, Any]:
            """
            Retrieve settings used to configure te embedding model in the specified type's index.
            """
            pass


Following the RAG architecture of an LLM being tacked onto a search engine,
this API proposes to split RAG functionality into a separate subclass.

Its role is to hold the search setup, as reflected in the constructor,
as well as to provide different ways to interact with an LLM. That
includes a direct passthrough, because, just as before, this is a handy
thing to have.

.. code-block:: python
    class GelRAG(GelAI):
        """
        This class stores a specific search configuration, a system prompt, and offers a way to access an LLM
        to generate answers to user's queries. Together those things make up a basic RAG system.
        """

        def __init__(
            self,
            system_prompt: str,
            search_type: Literal["semantic", "fulltext", "hybrid"],
            edgeql_query: str,
            shape: str,
            filter: str,
            group: str,
            limit: int,
            offset: int,
        ):
            """
            Args:
                system_prompt (str):        A prompt to pass to the LLM to let it know that it is, in fact, a RAG.
                search_type (str literal):  A keyword that determines what search method the RAG is going
                                            to use to do the retrieval.

            The rest of the arguments are mirrored from the search methods and are passed directly to them.

            Example:
                >>> gel_rag = GelRAG(
                        system_prompt="You are a RAG.",
                        search_type="semantic",
                        edgeql_query="Astronomy",
                        group=".solar_system",
                        limit=5,
                    )
                >>> gel_rag.query("how many moons does jupiter have", message_history=[])
            """
            pass

        def sync_query(
            self,
            text_query: str,
            message_history: List[str],
        ) -> str:
            """
            Query the RAG.
            """
            pass

        def stream_query(
            self,
            text_query: str,
            message_history: List[str],
        ) -> str:
            pass

        def sync_chat(
            self,
            text_query: str,
            message_history: List[str],
        ):
            """
            Bypass the retrieval step and generate a chat completion using the configured LLM.
            This method uses Gel as a passthough to the LLM provider API.
            """
            pass

        def stream_chat(
            self,
            text_query: str,
            message_history: List[str],
        ):
            pass

        @property
        def search_fn(self) -> Callable:
            """
            Get a partial that performs retrieval the way RAG was configured.
            """
            match self.search_type:
                case "semantic":
                    return partial(
                        self.semantic_search,
                        edgeql_query=self.edgeql_query,
                        shape=self.shape,
                        filter=self.filter,
                        group=self.group,
                        limit=self.limit,
                        offset=self.offset,
                    )
                case "fulltext":
                    return partial(
                        self.fulltext_search,
                        # same args as above
                    )
                case "hybrid":
                    return partial(
                        self.hybrid_search,
                        # same args as above
                    )

        @property
        def llm(self) -> Dict[str, Any]:
            pass


Extension Vectorstore Binding
----------------------------

The design of `GelVectorstore` follows that of the LangChain-LlamaIndex
integration. For it to be able to interact with embedding models (outside
of Gel) this binding also provides a simple model interface.

No assumptons are made about the raw content, meaning the user might wrap
CLIP into the interface and use it to generate and store image embeddings.

.. code-block:: python

    class BaseEmbeddingModel:
        """
        Interface for a callable that GelVectorstore is going to use
        to turn objects into embeddings.
        """

        def __call__(self, item: Any) -> List[float]:
            raise NotImplementedError

        @property
        def dimensions(self) -> int:
            pass

        @property
        def target_type(self) -> TypeVar:
            """
            Returns the type that the model embeds
            """
            pass


    class GelVectorstore:
        """
        This class provides a set of tools to interact with Gel's ext::vectorstore
        in a framework-agnostic way.
        It follows interface conventions commonly found in vector databases.
        """

        def __init__(
            self,
            embedding_model: BaseEmbeddingModel,
            collection_name: str = "default",
            record_type: str = "ext::ai::DefaultRecord",
        ):
            pass

        def add_item(self, item: Any, metadata: Dict[str, Any]) -> str:
            """
            Add a new record. The vectorstore is going to use it's embedding_model
            to generate an embedding and store it along with provided metadata.

            Returns the UUID of the inserted object.
            """
            pass

        def add_vector(self, vector: List[float], metadata: Dict[str, Any]) -> str:
            """
            Add a new record. The vectorstore is going to store the provided vector as is,
            as long as its dimensions match those configured in the schema.
            """
            pass

        def delete(self, ids: List[str]) -> List[Dict[str, Any]]:
            """
            Delete records by id. Return a list of deleted records, mirroring Gel's behaviour.
            """
            pass

        def get_by_id(self, id) -> Dict[str, Any]:
            """
            Get a record by its id.
            """
            pass

        def search_by_item(
            self,
            item: Any,
            metadata_filter: str,
            limit: int,
        ) -> List[Dict[str, Any]]:
            """
            Create an embedding for the provided item using the embedding_model,
            then perform a similarity search.
            """
            pass

        def search_by_vector(
            self,
            vector: List[float],
            metadata_filter: str,
            limit: int,
        ) -> List[Dict[str, Any]]:
            """
            Perform a similarity search.
            """
            pass

Modifications to the Gel Server
-------------------------------

To match the Python package changes, this RFC proposes the following
set of endpoints on the server side:

- ``semantic_search``
- ``fulltext_search``
- ``trigram_search``
- ``hybrid_search``

Parameters for of the endpoints in the search group would mirror those
from the Python methods:

- ``text_query``
- ``edgeql_query``
- ``shape``
- ``filter``
- ``group``
- ``limit``
- ``offset``

The following endpoints provide convenient access to external models
configured within the database:

- ``generate_embedding``
- ``generate_embedding_like``
- ``generate_llm_chat_completion``

This endpoint provides access to the end-to-end RAG workflow similar
to the current interface:

- ``query_rag``

Finally, these two endpoints offer a quick way to query settings for
models that are being used under the hood:

- ``get_embedding_model_config``
- ``get_llm_config``


Backwards Compatibility
=======================

The proposed implementation would break the existing interface, both on
the client side and the server side. This is caused by the difference in
the way it handles configuration and query parameters. Therefore,
maintaining backwards compatibility does not appear feasible.
