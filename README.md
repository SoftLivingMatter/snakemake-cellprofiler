# snakemake-cellprofiler

A thin wrapper around cellprofiler to simplify running pipelines on della.

## Usage

Install [mamba](https://github.com/conda-forge/miniforge?tab=readme-ov-file#install)
which is recommended for snakemake.

Create a snakemake environment, using version 7
```bash
mamba create -c conda-forge -c bioconda -n snake snakemake=7.32.4
```

Now you are ready to configure your run.  Fill out the following in `config/config.yaml`
```yaml
workdir: "/path/to/output/directory"
# set to a local cp installation if you need extra dependencies
# if unset, will use cellprofiler 4.2.6
cp_env: "cellprofiler"
# set to plugin directory
# if unset, will not include additional plugins
cp_plugin_dir: "/path/to/cellprofiler_plugins/"

# these can be left alone.  If you change, be sure to include the same wildcards
paths:
  config: "logs/config_{date}.yaml"
  slurm_output: "logs/slurm"

  batch_inputs: "{pipeline}/{image_set}/inputs/batch_{batch}.txt"
  batch_outputs: "{pipeline}/{image_set}/temp_outputs/batch_{batch}"
  final_outputs: "{pipeline}/{image_set}/outputs"

# this defines how to run each pipeline
pipelines:
  # these are the default resources for all jobs.  Can override below
  defaults:
    max_batch_size: 0.75  # total image size for batches, in GB
    # resources for cluster integration
    runtime: 63  # in minutes, keep >= 63
    mem_mb: 24000  # in MB
    continue_on_error: False  # if true, will skip images with errors
    recursive_images: False  # if true, include images in all subdirectories

  morphology:  # name replaces {pipeline} above
    pipeline_file: nucleolar_morphology.cppipe  # this is relative to the working directory
    mem_mb: 4000  # this overrides default memory for all image sets in this pipeline
    input_images:
      # name replaces {image_set}.  Location should have a {sample} wildcard
      set1:
        files: /scratch/gpfs/and/remember/{sample}.nd2
      set2:
        mem_mb: 16000  # in MB
        files: /scratch/gpfs/{sample}.nd2
      set3_or_another_name:
        mem_mb: 24000  # in MB
        runtime: 120  # in minutes, keep >= 63
        files: /scratch/gpfs/another/dir/{sample}.nd2
  another_pipeline:  # can define multiple pipelines
    pipeline_file: another_pipeline.cppipe
    input_images:
      # name replaces {image_set}.  Location should have a {sample} wildcard
      set2:  # can reuse names and paths with different pipelines
        mem_mb: 16000  # in MB
        files: /scratch/gpfs/{sample}.nd2
```

To test the resources you estimated, run
```bash
conda activate snake
snakemake --profile cluster test_resources
```
This will run the first 3 batches of each set.  You can examine the resource usage
with `reportseff` in the `logs/slurm` output directory.

**IMPORTANT** If you change the batch_size, you should delete the output files from
your test run!

To run the entire analysis, run `snakemake --profile cluster`.  If some jobs
fail, you can increase the runtime or memory and restart.  As long as the
batch size isn't changed, no files will be recreated.  It's a good idea to
run in a tmux session!
