### Notebook Discovery
Elsevier Data Wranglers and Data Scientists have been using Databricks for over 6 years.  During that time, we have created thousands of notebooks.  For years, we have tried to search for notebooks within our workspace but this has proven difficult due to the current limited Databricks search functionality.  For example, wouldn't it be great if you could restrict searches to a specific user's workspace folder, a notebook name, a specific command language, the command text for a specific cell, or a command submitted date range?   Wouldn't it be great if you could discover notebooks that have been cloned from an original notebook but where these cloned notebooks have been slightly modified?  We have seen similar feedback posted to the Databricks forums over the past couple of years (below are some representative examples).

- I'm updating a source table and need to find all the notebooks that have that table. I tried using the search function in databricks UI, but my problem is I'm getting results from every folder, including other users. Is there a way to conditionally limit to a certain folder?
- How can I search across multiple workspaces? I want to search about 10 different ones, and searching each one manually would be time-consuming.
- Can you search notebooks using conditionals such as exact phrase, contains(not), wildcards, or regular expressions?
- Is there possibility to check all notebooks from one particular Databricks workspace and get an inventory of where notebooks are directly accessing data through ADLS locations rather than through the metastore.
- Is there a way to search a text in all notebooks?  I like to find out whether a delta table is used in any notebooks.

To address these limitations and more, Elsevier Labs has open sourced Notebook Discovery.  

This is a 100% notebook solution making it easy to integrate this functionality into your current Databricks environment regardless of whether your language of preference is Scala, Python, SQL, or R.  Some users may find it useful to run this tool over their own user folder.  Groups may find it useful to run this tool over all of the user folders for the members of their group.  We particularly found this useful for our Labs group.  Since the tool is scalable, organizations may find it useful to run this tool for all of the user folders in their organization.  Whatever your decision, we hope you find the tool useful.

Darin McBeath<br/>
Elsevier Labs

##### Solution
Notebook Discovery consists of 3  notebooks.

- <i>NotebookIndex</i>
  - Creates the Notebook Discovery Index parquet file for all of the notebook information needed to power NotebookSearch and NotebookSimilarity. Each command in a notebook will be contained as a separate record in the index providing the capability to search at the command level (and not just the overall notebook).  For a more detailed description of the index structure that is generated, view the NotebookIndex notebook documentation.

- <i>NotebookSimilarity</i>
  - Compares the notebooks in the Notebook Discovery Index parquet file generated by NotebookIndex to produce a similarity score for every notebook.  We use the [MinHash](https://spark.apache.org/docs/latest/ml-features.html#minhash-for-jaccard-distance) for Jaccard distance algorithm to allow for scalability to tens of thousands of notebooks.  For more detailed information, view the NotebookSimilarity notebook documentation.

- <i>NotebookSearch</i>
  - Provides examples for searching the Notebook Discovery Index parquet file generated by NotebookIndex.  Included is a UDF to format the search results into HTML so the links to the notebook (or the specific command) can be quickly and easily followed.  For more detailed information, view the NotebookSearch notebook for more information.


##### Download

Notebook Discovery is provided as a [download](https://github.com/elsevierlabs-os/NotebookDiscovery/tree/main/download/NotebookDiscovery-1.0.0.dbc).  This is a DBC (Databricks archive) file.


##### Installation

From the Databricks UI, simply import the Notebook Discovery archive downloaded above into the workspace folder where you want to place Notebook Discovery. The workspace folder where the archive is imported is unrelated to the workspace folders that you want to index.

##### Execution

In addition to the 3 notebooks discussed above, there are also a couple of run notebooks that have been created to simplify using Notebook Discovery.  Using the run notebooks, you can simply adjust the parameters required of the notebook and run the notebook.  While the run notebooks are implemented in scala, they are straightforward and should be usable by users familiar with other programming languages.  

- NotebookIndexRun
  - The first run notebook to execute is NotebookIndexRun to generate the Notebook Discovery Index parquet file which the subsequent notebooks require.  After NotebookIndexRun completes successfully, you can run either  NotebookSimilarityRun or model queries contained in NotebookSearch.
  - Arguments include the following:
    - path. The location of the notebook to run (in this case NotebookIndex).  It is assumed this notebook is in the same folder as it's run notebook.
    - timeoutSeconds.  Number of seconds to allow for the notebook execution (currently set at 2 hours).
    - folders.  Comma separated list of workspace folders to crawl when generating the Notebook Discovery Index.
    - indexFilename. Where to place the generated  Notebook Discovery Index parquet file.  Typically an area mounted on Databricks.
    - overwrite. Whether the Notebook Discovery Index parquet file should be overwritten if it already exists.  'False' will not overwrite.  'True' will overwrite.
    - parallelization. The number of concurrent requests that will be sent to Databricks through the REST APIs when creating the index (currently set at 8).
  - Below is a sample request.
```
dbutils.notebook.run(path="NotebookIndex",
                              timeoutSeconds=7200,
                              arguments=Map("folders" -> """
                                                            /Users/someone1@elsevier.com/,
                                                            /Users/someone2@elsevier.com/
                                                            """,
                                            "indexFilename" -> "/mnt/some-mount-point/DatabricksNotebookSearch/index",
                                            "overwrite" -> "False",
                                            "parallelization" -> "8"))                                            
```

- NotebookSimilarityRun
  - If you desire to generate similarity scores for the notebooks, execute NotebookSimilarityRun</a> to generate the Notebook Discovery Similarity file.  After NotebookSimilarityRun completes successfully, you can analyze the resulting parquet file and explore similar notebooks.
  - Arguments include the following:
    - path. The location of the notebook to run (in this case NotebookSimilarity).  It is assumed this notebook is in the same folder as it's run notebook.
    - timeoutSeconds.  Number of seconds to allow for the notebook execution (currently set at 2 hours)
    - indexFilename. Location of the Notebook Discovery Index parquet file generated by NotebookIndex.
    - similarityFilename. Where to place the generated  Notebook Discovery Similarity parquet file  Typically an area mounted on Databricks.
    - overwrite. Whether the Notebook Discovery Similarity parquet file should be overwritten if it already exists.  'False' will not overwrite.  'True' will overwrite.
    - similarDistance. Max Jaccard distance to use when comparing notebooks
    - vocabSize. Number of unique words (ngrams) you would expect to occur across all of the notebooks that were indexed.
    - ngramSize. Size of the ngram to use when tokenizing and comparing notebooks
    - minDocFreq. Number of notebooks an ngram must occur before it is included in the vocabulary
  - Below is a sample request.
```
dbutils.notebook.run(path="NotebookSimilarity",
                              timeoutSeconds=7200,
                              arguments=Map("indexFilename" -> "/mnt/some-mount-point/DatabricksNotebookSearch/index",
                                            "similarityFilename" -> "/mnt/some-mount-point/DatabricksNotebookSearch/index-similarity",
                                            "overwrite" -> "False",
                                            "similarDistance" -> ".25",
                                            "vocabSize" -> "5000000",
                                            "ngramSize" -> "5",
                                            "minDocFreq" -> "1"))                                            
```

- NotebookSearch
  - If you desire to find notebooks or commands that contain specific text, then look at NotebookSearch for examples.  Since the Notebook Discovery Index is a parquet file, basic filter and contains operations can be used to identify notebooks and commands satisfying your query criteria.   
  - Below is a simple example that searches notebooks where the language is 'scala', the folder starts with  '/Users/d.mcbeath@elsevier.com/' and the command text contains 'udf' and displays 10 results.
```
  val notebookCommands = spark.read.parquet(indexFilename)
  displaySearchResults(notebookCommands.filter($"cLang" === "scala")
                                       .filter($"cText".contains("udf"))
                                       .filter($"nbFolder".startsWith("/Users/d.mcbeath@elsevier.com/")),num=10)                                       
```

##### Assumptions
- Notebook Discovery Index and Similarity files are currently written as a parquet files.  If a desire is to use Databricks tables, then one could easily add another step to the Run jobs to create a table from the parquet files or adjust the underlying code to write tables instead of a parquet files.

##### Limitations
- If a notebook name contains a "/" character, this notebook will currently be ignored.
- If an exported notebook contents exceeds (10 MB), the notebook will not be processed.
- The Notebook Discovery Index parquet file is a static point in time snapshot.  If your notebooks are updated regularly, then you may want to rerun the NotebookIndexRun on a regular basis (such as weekly).
- The notebooks to be indexed by Notebook Discovery must be stored in the Databricks workspace and not an external repository such as github.
- The notebooks to be indexed by Notebook Discovery must have read permissions for the user that runs NotebookIndexRun.
- In order to view the notebooks (and commands) linked from NotebookSearch, users will need to have read permissions for those notebooks.

##### Future Plans
- We plan to record the user/group permissions for each notebook in the Notebook Discovery Index parquet file.  This would allow filtering on the permissions associated with a notebook.  This information could also be used to filter the NotebookSearch results to only those that are accessible by a user.
- Improve parallelization.  Currently, if only one folder is specified in the folders argument for NotebookIndexRun, there is no parallelization.
