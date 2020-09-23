# OpenWebText2

This repository is part of Eleuther AI's quest to create a massive repository of high quality text data for training language models.

## Background

OpenAI required around 40gb of high quality text corpus For training [GPT2](https://openai.com/blog/better-language-models/). Common Crawl provides the scale necessary for modern language models, however the quality is unreliable. Manual curation of Common Crawl is always an option, albeit an expensive one. Thankfully Reddit provides high quality decentralized curation by design, and this became the key innovation for the WebText dataset.

The generation of WebText can be summarized as:
1. Scrape URLs from all Reddit submissions up to and including December 2017 with 3 or higher score.
2. Deduplicate scraped content based on URL
3. Exclude wikipedia (OpenAI already had a separate Wikipedia dataset)
4. Deduplicate remaining content using undisclosed "heuristic based cleaning". This includes removal of non-english web pages.

Neither the resulting corpus or generation source code was made public, inspiring Aaron Gokaslan and Vanya Cohen to create the [OpenWebTextCorpus](https://skylion007.github.io/OpenWebTextCorpus/).

OpenWebTextCorpus is an open source reproduction of WebText, reifying the "heuristic based cleaning" stage using fuzzy deduplication enforcing a minimum token length. For content based de-duplication they used local-sensitivity-hashing (LSH) with minhash on sets of 5-grams at the document level. Documents were then tokenized and any with less then 128 tokens were removed. After all processing there remained 40GB of text across 8,013,769 documents. The original code is unavailable at this time, but there are several popular repositories that cover the pipeline to various degrees.

## OpenWebText2 Motivation

Our primary goals for the corpus are:

1. More data. Coverage of the original OpenWebTextCorpus ended at December 2017.
2. Include all languages, providing metadata for easy filtering
3. Provide several version of the generated corpus for differing user requirements:
    * Raw version containing all scraped urls from reddit submissions with associated reddit metadata
    * Plug and play version based on submissions of minimum 3 score with content based fuzzy de-duplication
    * For both of the above versions, the Corpus will be broken up by month/year of Reddit submission and then frozen. Going forward we will provide processed corpus for additional months as they become available on PushShift.
4. Provide full source code for all stages of the pipeline including deduplication.

Some additional stretch goals:
1. PyTorch dataset
2. TensorFlow dataset

We decided on a rewrite taking inspiration from both https://github.com/yet-another-account/openwebtext/ and https://github.com/jcpeterson/openwebtext.

As mentioned, once we have completed generating the dataset it will be available for download, broken down by month/year of the original Reddit submission. Going forward we will also be providing the corpus as 

## The Pipeline

[PushShift](https://www.reddit.com/r/pushshift/comments/bcxguf/new_to_pushshift_read_this_faq/) provides dumps of all reddit posts and submissions, however they are normally a few months behind. While this would be problematic for certain use cases, we don't require up to the minute data for training GPTNeo. For the initial stage of this project we decided to avoid scraping more recent Reddit submissions either directly or via APIs. We may add this in the future.

1. Download Reddit submission dump files from PushShift
2. Process the files to extract URLs from all non-self submissions. Save URLs and Reddit metadata with [lm_dataformat](https://github.com/leogao2/lm_dataformat)
3. Deduplicate the URLs
4. Scrape the URLs using [Newspaper3k](https://newspaper.readthedocs.io/en/latest/), saving both text and metadata using lm_dataformat
5. Perform fuzzy deduplication using [MinHashLSH](http://ekzhu.com/datasketch/lsh.html)

**Acknowledgements**  
Much of this code was written by @researcher2, with inspiration and some straight copying of the scraping code found at https://github.com/yet-another-account/openwebtext/. @sdtblck kindly put together the Colab notebook.

# Processing Status
We are currently attempting to scrape all of 2011 up to the most recent pushshift dump. Deadline is 30/09/2020, current status:

| Year       | Months | Responsible         | Status     |
| :--------: | :----: | :-----------------: | :--------: |
|  2011      |        | researcher2         | Done!    |
|  2012      |        | researcher2         | Done!    |
|  2013      |        | researcher2         | Done!    |
|  2014      |        | bmk                 | currently running on nuck |
|  2015      |        | researcher2         | Running on Eleuther Hetzner. August currently running |
|  2016      |        | researcher2         | Used free credits for google cloud 8 vcpu. September currently running|
|  2017      | 1-5    | Sid                 | Done! |
|   -        | 6      | Stephen5311         | Stephen has started on June on his beast machine at home. |
|   -        | 7-11   | Sid                 | Running on colab |
|   -        | 12     | Unassigned          | Unassigned |
|  2018      | 1-3    | researcher2         | Created a second Hetzner box, March currently running  |
|   -        | 4-12   | bmk/researcher2     | Whoever makes it here first, researcher2 going forward, bmk going back |
|  2019      | 1      | researcher2         | Done!    |
|   -        | 2-12   | bmk                 | feb-dec on nuck    |
|  2020      | 1-4    | researcher2         | Running on colab. Jan & Feb complete. |

# Process in colab:

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/EleutherAI/pushshift_dump_processing/blob/master/webtext2_colab.ipynb)

Open the above colab notebook. We recommend you run 5 Colab instances at once, each doing a single dump file. For each instance, set the start and end date/month to be the desired month/year, then change the "colab_instance" to 1 for the first instance, 2 for the second etc. Run the first few cells manually as the Google Drive requires a manual authorization key. After that simply run all remaining cells to perform the dump download, url extraction and url scraping steps.

This may take over a day, so you will need to resume after Colab kicks you off. Re-run the setup setups, and then the final cell. The scraper script called in the final cell will resume from a checkpoint file saved in your google drive. It is important that the colab_instance remains the same, so I save one copy of the Colab notebook to my google drive for each instance. You could just leave it running, but you might be confused if your computer crashed!

When this is finished, copy the final files over to your drive, and you're done!

# Process locally:

## Environment Setup
Tested in a basic conda environment, though you could use venv or even the global python environment if you wish. I use miniconda to avoid a bloated download.

**Miniconda Install For Linux**

Follow the below steps, or read the conda instructions if you wish: https://docs.conda.io/projects/conda/en/latest/user-guide/install/linux.html

```bash
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
sha256 Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh
```
Select yes on the init step.

Restart your shell to refresh the path.

**Create and activate conda environment**

Environments are saved in a central store on the local disk, no need to create folders like with venv.
```
conda create --name pushshift python=3.8
conda activate pushshift
```

**Install Repo and Requirements**
```bash
git clone https://github.com/EleutherAI/pushshift_dump_processing
cd pushshift_dump_processing
pip install -r requirements.txt
```

## Overall Summary
There are three parts in this pipeline:

1. Download the compressed pushshift dumps
2. Process the downloaded dump files
3. Scrape the URLs sourced from step 2


## Part 1 - Downloading Compressed Dump Files

This is done within **download_pushshift_dumps.py**.

As the pushshift dumps shifted between various compressions formats the program cycles through all possibilities until it finds a matching file for a given month, preventing any duplicates from being downloaded.

To run, either change the hardcoded parameters inside of __name__ == '__main__', or call the main method from another program. 

A date range on the main method allows you to do a single month at a time for a concurrent pipeline or if disk space is an issue.

The following example will download all pushshift dumps up until the most recent into the *dumps* subfolder of the current path. Log can be found in *./download_pushshift_dumps.log*.

```python
if __name__ == '__main__':
    logfile_path = "download_pushshift_dumps.log"
    
    start_date = datetime.datetime(2011, 1, 1)    
    end_date = datetime.datetime.now()

    download_directory = "dumps"

    main(logfile_path, start_date, end_date, download_directory) 
```
 

## Part 2 - Processing The Downloaded Dumps

This is done within **process_dump_files.py**.

Either change the hardcoded parameters inside of __name__ == '__main__', or call the main method from another program. 

The following example will process all dump files found within the *dumps* directory - filename matching "RS_*" for the relevant compression formats. For each matching file a separate *name.stats.json* and *name.urls.txt* will be created in the output directory.

Currently we support the three different compression formats provided by PushShift - bz2, xz, zst.

```python
if __name__ == '__main__':
    dumps_directory = "dumps"
    output_directory_root = "output"

    main(dumps_directory, output_directory_root)
```

## Part 3 - Scraping From Sourced URLs

This is done within **scrape_urls.py**. 

Either change the hardcoded parameters inside of __name__ == '__main__', call the main method from another program, or use command line arguments.

This program iterates through a URL file generated in step 2 above. It loads batches of URLs and hands them out to worker processes which scrape using newspaper scraper. Each batch will be archived using jsonl zst provided by [lm_dataformat](https://github.com/leogao2/lm_dataformat)
(thanks @bmk). Some metadata like language, url, top level domain, word count, and title are saved in the metadata field offered by lm_dataformat.

You may need to modify the batch size and process count depending on your environment. The default settings are batch size 10000, and process count 60, this will spike cpu usage to 100% at the start of each batch. 

If you want to filter the urls before scraping we have left an example filter in **filter.py**. This is mainly to speed up the process by avoiding timeouts or files that obviously won't contain text.

NOTE: The program saves a checkpoint file in the same directory as the url.txt file to allow you to resume later if the job dies or needs to be killed. **DON'T** change the batch size if resuming an existing run.

The following example will scrape all URLs found in *output/RS_2011-01.urls.txt*, using lm_dataformat to save the text and metadata into files within *scrapes/rs_2011-01*.

```bash
python scrape_urls.py output/RS_2011-01.urls.txt scrapes
```
