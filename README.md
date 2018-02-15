# Citeomatic

This is the source distribution for the [Citeomatic](http://labs.semanticscholar.org/citeomatic/) service and for the paper **Content-based Citation Recommendation**.

## Clone the repo
```
git clone git@github.com:allenai/citeomatic.git
```

## Install direnv (Optional)
Citeomatic uses [direnv](https://github.com/direnv/direnv) to activate the `ai2-citeomatic` conda environment whenever you `cd` into the root directory. Alternatively, you can skip this step and activate the conda environment manually yourself.
 
* Install `direnv`:
	* Ubuntu: 
		```
		sudo apt-get install direnv
		```
	* OSX:
		```
		brew install direnv
		```
	Then:
		```
		cd citeomatic/ ;
		direnv allow .
		```


## Setup

Citeomatic uses the tensorflow framework to run its neural network models. It was tested on NVIDIA 
GPUs (GeForce GTX 1080 and Tesla V100-SXM2). Please install the required GPU drivers for your GPU
. 

Once the required drivers are installed: 

```bash
./env.sh
```
The script will setup a conda environment (`ai2-citeomatic`), the citeomatic package and its dependencies. 

Verify that you now have the `ai2-citeomatic` conda environment installed and activated. If not activated, `source activate ai2-citeomatic`.

### Common Issues
  1. If you see this error when running any of our scripts:
```
ImportError: libcusolver.so.8.0: cannot open shared object file: No such file or directory
```
please set the environment variable: `export LD_LIBRARY_PATH=/usr/local/cuda/lib64/`

  2. If you have multiple GPUs, please set the environment variable `export 
  CUDA_VISIBLE_DEVICES=<gpu number>`. Citeomatic does not use more than 1 GPU but tensorflow will
   spawn a process on all available GPUs. 

## Download data
```
./get-data.sh <location> 
```
The script downloads all the required data and trained models to the provided location and adds a symlink from a local `data` directory to the provided `<location>`. Alternatively, you can provide `data/` as the location to avoid the symlink.

**IMPORTANT**: 

If you have access to the AI2 Corp network, you can set `location` to  `/net/nfs.corp/s2-research/citeomatic/public/` to avoid duplicating data.

## Citeomatic Evaluation

This section details how to run the end-to-end system using pre-trained models
and evaluate performance of Citeomatic for each dataset. If you successfully executed the previous steps, trained models should already be available.

 * Open Corpus
```bash
python citeomatic/scripts/evaluate.py --dataset_type oc --candidate_selector_type ann --split test --paper_embedder_dir data/open_corpus/models/paper_embedder/ --num_candidates 5 --ranker_type neural --citation_ranker_dir data/open_corpus/models/citation_ranker/ --n_eval 20000
```  

 * Pubmed
```bash
python citeomatic/scripts/evaluate.py --dataset_type pubmed --candidate_selector_type ann --split test --paper_embedder_dir data/comparison/pubmed/models/paper_embedder/ --num_candidates 10 --ranker_type neural --citation_ranker_dir data/comparison/pubmed/models/citation_ranker/

```

 * DBLP
```bash
python citeomatic/scripts/evaluate.py --dataset_type dblp --candidate_selector_type ann --split test --paper_embedder_dir data/comparison/dblp/models/paper_embedder/ --num_candidates 10 --ranker_type neural --citation_ranker_dir data/comparison/dblp/models/citation_ranker/
```

## BM25 Baseline

 * Open Corpus
```bash
python citeomatic/scripts/evaluate.py --dataset_type oc   --candidate_selector_type bm25 --split test --ranker_type none --num_candidates 5
```  

 * Pubmed
```bash
python citeomatic/scripts/evaluate.py --dataset_type pubmed   --candidate_selector_type bm25 --split test --ranker_type none --num_candidates 10
```

 * DBLP
```bash
python citeomatic/scripts/evaluate.py --dataset_type dblp   --candidate_selector_type bm25 --split test --ranker_type none --num_candidates 10
```

## Train.py
The main script to train and tune hyperparameters for various models is `train.py`. Usage:

```bash
python train.py [options]
```

  * General Parameters:
	  * `--mode` (Required): The mode to run the `train.py` script in. Possible values: `train` or 
	  `hyperopt`. The `train` mode will train a single model and save to a given location. The 
	  `hyperopt` mode will run hyperparamter-optimization and return the best found model.
	  * `--dataset_type`: Dataset to use. Possible values: `dblp` (default), `pubmed` or `oc`
	  * `--model_name`: Possible values: `paper_embedder` (default) or `citation_ranker`
	  
  * Parameters specific to Hyperparameter Optimization
	  * `--max_evals_initial`: No. of models to train in the first phase. Our hyperparameter 
	  optimization method runs in two steps. In the first step, a large number of models are run 
	  for a few epochs and the best performing 10 models are run for more number of epochs in the
	   second phase.
	  * `--max_evals_secondary`: No. of models to train in the second phase. Best 
	  `max_evals_secondary` models from Phase 1 are trained for a longer time
	  * `--total_samples_initial`: No. of samples to train first phase models on
	  * `--total_samples_secondary`: No. of samples to train second phase models on
	  * `--models_dir_base`: Base directory to store hyperopt results in 
	  * `--n_eval`: No. of validation examples to evaluate a trained model
	  * `--run_identifier`: A string to identify the experiment
	  * `--version`: Version string to be appended to the directory used to store model in
  
  * Parameters specific to Training a single model
      * `hyperopts_results_pkl`: Path to the `.pkl` file generated by the hyperopt mode
      * `options_json`: Optional json file containing all options required to train a model
      
    Refer to the `ModelOptions` class for more options.

### Hyper-parameter optimization

We use the [hyperopt](https://github.com/hyperopt/hyperopt) package to tune hyperparameters. 
Use the following command to run the hyperopt on a particular dataset

  * Paper Embedder Model 

```bash
python citeomatic/scripts/train.py   --mode hyperopt   --dataset_type <dataset> --n_eval 500 --model_name paper_embedder   --models_dir_base  data/hyperopts/<dataset>/ --version <version> &> data/hyperopts/dblp/dblp.paper_embedder.hyperopt.log
```

  * Citation Ranker Model
  

## Miscellaneous

  1. **Create a new BM25 Index**
  
  The `get-data.sh` script also downloaded a pre-built BM25 indexe for each dataset (pubmed, dblp and oc). But, a new index can be built as:
  
```
python citeomatic/scripts/create_bm25_index.py --dataset_name <dataset name> 
```
	
  Modify `CreateBM25Index` to change the way the BM25 index is built. We use the [whoosh](https://pypi.python.org/pypi/Whoosh/) package to build the BM25 index. To change the way the index is queried, change the `fetch_candidates` implementation in `BM25CandidateSelector`

  This script will create an index at this location: `data/bm25_index/<dataset name>/`


  2. **Re-Create SQLite DB for dataset**
  
  This following scripts will create an index at this location: `data/db/<dataset name>.sqlite.db`

  * For the DBLP and Pubmed datasets:
	```
	python citeomatic/scripts/convert_kdd_to_citeomatic.py --dataset_name <dataset name>
	```

  * For the open corpus dataset:
	```
	python citeomatic/scripts/convert_open_corpus_to_citeomatic.py
	```

The SQLite DB is used to speed-up retrieving documents for a particular document id. 

## Team

Citeomatic is an open-source project backed by the [Allen Institute for Artificial Intelligence (AI2)](http://www.allenai.org).
AI2 is a non-profit institute with the mission to contribute to humanity through high-impact AI research and engineering.
To learn more about who specifically contributed to this codebase, see [our contributors](https://github.com/allenai/citeomatic/graphs/contributors) page.