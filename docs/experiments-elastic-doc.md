# Anserini: Multi-field BM25 Baselines for MS MARCO Document Ranking

This page contains instructions for reproducing the Elasticsearch optimized
multi_match best_fields" entry on the the [MS MARCO Document Ranking Leaderboard](https://microsoft.github.io/MSMARCO-Document-Ranking-Submissions/leaderboard/).

This run makes sure to preserve the distinction between document fields when
preparing and indexing documents. For ranking, we use a disjunction max query to
combine score contributions across fields. The weights for the disjunction max
query are taken from the "multi_match best_fields tuned (all-in-one): all
params" entry in the [blog post](https://www.elastic.co/blog/improving-search-relevance-with-data-driven-query-optimization)
that describes the leaderboard submission.

To match the leaderboard results, this run makes use of a custom stopwords file
'msmarco-stopwords.txt'. The file contains the default English stopwords from
Lucene, plus some additional words targeted at question-style queries.

## Data Prep

We're going to use the repository's root directory as the working directory.
First, we need to download and extract the MS MARCO document dataset:

```
mkdir collections/msmarco-doc
wget https://msmarco.blob.core.windows.net/msmarcoranking/msmarco-docs.tsv.gz -P collections/msmarco-doc
gunzip collections/msmarco-doc/msmarco-docs.tsv.gz
```

To confirm, `msmarco-docs.tsv.gz` should have an MD5 checksum of `103b19e21ad324d8a5f1ab562425c0b4`.

First we need to convert the file to JSON lines format. Each document will
correspond to a JSON object with distinct fields for title, URL, and body:

```
python tools/scripts/msmarco/convert_doc_collection_to_jsonl.py \
  --collection-path collections/msmarco-doc/msmarco-docs.tsv \
  --output-folder collections/msmarco-doc-json
```

We then build the index with the following command:

```
sh target/appassembler/bin/IndexCollection -threads 4 -collection JsonCollection \
  -generator DefaultLuceneDocumentGenerator -input collections/msmarco-doc-json/ \
  -index indexes/msmarco-doc/lucene-index-msmarco -storeRaw \
  -stopwords msmarco-stopwords.txt
```

On a modern desktop with an SSD, indexing takes around 15 minutes.
There should be a total of 3,201,821 documents indexed.

## Performing Retrieval on the Dev Queries

After indexing finishes, we can do a retrieval run. A few minor details to pay
attention to: the official metric is MRR@100, so we want to only return the top
100 hits, and the submission files to the leaderboard have a slightly different
format.

```bash
sh target/appassembler/bin/SearchMsmarco -hits 100 -threads 4\
  -index indexes/msmarco-doc/lucene-index-msmarco/ \
  -queries src/main/resources/topics-and-qrels/topics.msmarco-doc.dev.txt \
  -output runs/run.msmarco-doc.leaderboard-dev.bm25base.txt \
  -stopwords msmarco-stopwords.txt \
  -k1 1.2 -b 0.75 -fields contents=10.0f -fields title=8.63280262513067f \
  -dismax -dismax.tiebreaker 0.3936135232328522f
```

On a modern desktop with an SSD, the run takes around 6 minutes.

After the run completes, we can evaluate the results:

```bash
python tools/scripts/msmarco/msmarco_doc_eval.py --judgments src/main/resources/topics-and-qrels/qrels.msmarco-doc.dev.txt --run runs/run.msmarco-doc.leaderboard-dev.bm25base.txt
#####################
MRR @100: 0.3071939335136768
QueriesRanked: 5192
#####################
```

## Replication Log

+ Results replicated by [@jtibshirani](https://github.com/jtibshirani) on 2020-02-26 (add commit here)
