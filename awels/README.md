```markdown
# Shopizer Knowledge Base

- **Repository:** [shopizer on GitHub](https://github.com/fbeawels/shopizer.git)

## Project Overview

This project leverages advanced data processing techniques to create a comprehensive knowledge base for the Shopizer repository. Using state-of-the-art tools for code, document, and image analysis, this knowledge base enhances accessibility and understanding of the Shopizer project by providing context, prompts, and specifications.

## Processing Summary

In creating this knowledge base, various files from the Shopizer repository were analyzed and processed to form structured collections of data. These collections are aimed at assisting developers and contributors who interact with the Shopizer codebase.

Categories and file counts processed:

- **Code files:** 1,237 
- **Documentation files:** 3
- **Image files:** 43

## Tools Used

The following tools and technologies were used to process the repository and create the knowledge base:

- **LLM:** OpenAI GPT-4o for generating context and analyzing code
- **Embeddings:** Ollama with the `nomic-embed-text` model for creating vector representations
- **Vector Database:** Qdrant for storing and querying vectorized data
- **Code Analysis:** `build_code.py` script for code file processing
- **Document Analysis:** `build_doc.py` script for documentation processing
- **Image Analysis:** `build_multi.py` script for image file processing

## Statistics

The processed data was organized into three main vector database collections, each containing several data points:

| Collection Name                          | Points  |
|------------------------------------------|---------|
| Code Collection (`fbeawels-shopizer-code`) | 1,988   |
| Documentation Collection (`fbeawels-shopizer-doc`) | 90      |
| Image Collection (`fbeawels-shopizer-multi`) | 43      |

## Generated Files

The following files were generated to assist developers and AI agents in understanding and interacting with the knowledge base:

- **`CONTEXT.md`:** Provides detailed context about the Shopizer repository.
- **`PROMPT.md`:** Contains the system prompt that can be used with AI tools.
- **`SPECS.md`:** Includes specifications for creating a Langflow agent.

## Usage Instructions

To interact with the knowledge base and utilize the generated data:

1. **Clone the Repository:**
   ```
   git clone https://github.com/fbeawels/shopizer.git
   ```

2. **Explore the Context and Prompts:**
   - Review the content in `CONTEXT.md` for detailed information about the repository.
   - Use `PROMPT.md` to understand and apply the AI system prompts.

3. **Access the Vector Database:**
   - Connect to the Qdrant vector database to query the processed collections.
   - Utilize the collections to gain deeper insights and extract relevant information for development or research purposes.

This comprehensive framework aims to simplify the process of understanding and contributing to the Shopizer project by organizing and presenting key information effectively.

For any further questions or contributions, feel free to open an issue or submit a pull request to the repository.

---
End of README.
```
